# Deploy API — SSH-Signature-Authenticated Deploy Trigger

**Purpose:** lets ddashboard (or any other trusted caller) remotely trigger a production deploy (currently `advoapp-production`) over HTTP, with real-time status and commit tracking, without a shared-secret bearer token.

**Live at:** `https://stella-deployment-api.foxcraft.digital`

**Why SSH-signature auth instead of a bearer token/API key:** reuses SSH's existing trust model. ddashboard already has SSH keys; instead of minting and managing a separate API secret (rotation, secure storage, risk of leaking in logs), the caller **signs a short-lived payload** with its own SSH private key using OpenSSH's native `ssh-keygen -Y sign`/`-Y verify` (the same mechanism used for signed git commits) — the private key itself never leaves ddashboard, never crosses the network.

**Every endpoint that returns non-public information requires a valid signature** — `/apps` and `/deploy/*` alike. Only `/health` is open, since it reveals nothing beyond "the process is up."

---

## Architecture

Containerized, on the `edge` Docker network (no published host port — same pattern as every other Stella service). Runs as a FastAPI app under Uvicorn.

**Why containerized rather than a native systemd service (as first attempted):** a systemd-native version was built and then abandoned mid-setup once it became clear it broke the "everything routes through Caddy on `edge`, zero new firewall work per service" principle established earlier — it would have needed its own bespoke UFW rule. Containerizing it, with the Docker socket mounted in so it can still launch sibling build containers, keeps it consistent with the rest of the stack.

```yaml
# /opt/services/docker-compose.yml (relevant service)
deploy-api:
  build: ./deploy-api
  container_name: deploy-api
  env_file:
    - ./azure-deploy/.env
  volumes:
    - /var/run/docker.sock:/var/run/docker.sock
    - /opt/apps:/opt/apps
    - /opt/services/azure-deploy:/opt/services/azure-deploy
    - ./deploy-api/allowed_signers:/opt/services/deploy-api/allowed_signers:ro
    - ./deploy-api/logs:/opt/services/deploy-api/logs
    - /root/.ssh:/root/.ssh:ro
  networks:
    - edge
  restart: unless-stopped
```

**`/root/.ssh:/root/.ssh:ro` is essential** — the container needs the same SSH key + `known_hosts` Stella's host already uses to clone/pull the private GitHub repo. Without this mount, the deploy script's `git pull` step fails inside the container with `Host key verification failed` / `Could not read from remote repository`, even though the exact same script works fine when run directly on the host (which already has SSH trust established). Hit and fixed during initial setup.

**Dockerfile** (`/opt/services/deploy-api/Dockerfile`) installs: `git`, `zip`, `openssh-client` (needed for `ssh-keygen -Y verify`), Docker CLI (to launch sibling containers via the mounted socket), and the Azure CLI (so the mounted `deploy-advoapp.sh` script can call `az` from inside this container).

---

## Auth mechanism

**Allowed signers file** (`/opt/services/deploy-api/allowed_signers`):
```
ddashboard ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIOpVG5e5AzGVVKU9uIWEACauLbP/w+SOTw3MnSPqruVn foxcraft-digital@dedi6724
```
Format: `<principal> <key-type> <base64-key>`. `ddashboard` is the "principal" name checked against on every verification call.

**Payload format:** `<action>:<unix_timestamp>`. `action` identifies what's being authorized:
- `list-apps` for `POST /apps`
- `deploy-<app_name>` for `POST /deploy/{app_name}`, e.g. `deploy-advoapp-production`

The timestamp provides replay protection — rejected if more than 60 seconds old or in the future (`REPLAY_WINDOW_SECONDS`).

**Signing (caller side, e.g. on ddashboard):**
```bash
TIMESTAMP=$(date +%s)
PAYLOAD="deploy-advoapp-production:${TIMESTAMP}"   # or "list-apps:${TIMESTAMP}"
echo -n "$PAYLOAD" > /tmp/payload.txt
ssh-keygen -Y sign -f ~/.ssh/id_ed25519 -n deploy /tmp/payload.txt
```
Produces `/tmp/payload.txt.sig`. `-n deploy` sets the "namespace," which must match the API's `NAMESPACE = "deploy"` constant on both sides — used across every action, not just deploys.

**Verification (server side)** — shared by every authenticated endpoint via one helper:
```python
def verify_request(action: str, req: DeployRequest, request: Request) -> None:
    """Raises HTTPException if the signed request is invalid."""
    expected_prefix = f"{action}:"
    if not req.payload.startswith(expected_prefix):
        raise HTTPException(400, "payload does not match expected action")
    if not check_replay(req.payload):
        raise HTTPException(401, "stale or invalid timestamp")
    if not verify_signature(req.payload, req.signature):
        raise HTTPException(401, "signature verification failed")
```
`verify_signature` itself shells out to `ssh-keygen -Y verify -f allowed_signers -I ddashboard -n deploy -s <sigfile>` against the payload text.

**Sending a request:**
```bash
curl -s -X POST https://stella-deployment-api.foxcraft.digital/apps \
  -H "Content-Type: application/json" \
  -d "$(python3 -c "
import json
payload = open('/tmp/payload.txt').read()
sig = open('/tmp/payload.txt.sig').read()
print(json.dumps({'payload': payload, 'signature': sig}))
")"
```
Same pattern for `/deploy/{app_name}`, just with the matching `action` string signed beforehand.

---

## Endpoints

| Method | Path | Auth | Purpose |
|---|---|---|---|
| GET | `/health` | none | Health check |
| POST | `/apps` | signed (`list-apps`) | List available app names to deploy |
| POST | `/deploy/{app_name}` | signed (`deploy-{app_name}`) | Trigger a deploy. Returns `{"job_id": "...", "status": "queued"}` |
| GET | `/deploy/{app_name}/status/{job_id}` | none¹ | Poll current job state |
| GET | `/deploy/{app_name}/status/{job_id}/stream` | none¹ | Server-Sent Events — real-time log tail, ends with a `done` event |
| GET | `/deploy/{app_name}/latest` | none¹ | Most recent job for that app |

¹ Status/stream/latest endpoints aren't signature-gated — a `job_id` is an unguessable UUID, so knowing one already implies you (or something acting on your behalf) triggered or was told about that specific deploy. Consider adding auth here too if job IDs ever get logged/exposed somewhere less trusted.

**App allowlist** (`APPS` dict) — only apps explicitly listed can be triggered or seen via `/apps`; `app_name` is never used to build an arbitrary filesystem path:
```python
APPS = {
    "advoapp-production": {
        "script": "/opt/services/azure-deploy/deploy-advoapp.sh",
        "src_dir": "/opt/apps/dotnet/advoapp-production.finditoo.foxcraft.digital/src",
    }
}
```
Adding a future app = one new entry, pointing at its own deploy script and source checkout. No other code changes needed.

**Job tracking:** in-memory only (a Python dict guarded by a lock) — lost on container restart. A persistent plain-text audit trail still exists separately via `deploy.log` (`LOG_PATH`), which records every accepted/rejected request and every finished job with its outcome and commit SHA.

---

## Commit tracking

Every job response includes `commit_sha` and `commit_message` — the actual commit that was checked out and deployed, captured directly from the repo right after `git pull` completes (not just whatever HEAD was on the remote at trigger time, in case something pushes mid-deploy):

```python
def get_git_info(src_dir: str):
    result = subprocess.run(
        ["git", "log", "-1", "--format=%H|||%s"],
        cwd=src_dir, capture_output=True, text=True, timeout=10,
    )
    sha, _, message = result.stdout.strip().partition("|||")
    return sha, message
```

Captured the moment the deploy script's output reaches its `2/5` marker (publish step starting) — a reliable signal that `1/5` (pull) just finished successfully. Fields are `None` until that point; a `status: "failed"` job whose failure happened during step 1 will have `commit_sha: null`.

Example job response once populated:
```json
{
  "app": "advoapp-production",
  "status": "success",
  "current_step": "5/5 — Deploying to Azure App Service...",
  "commit_sha": "a3f2c91...",
  "commit_message": "Fix invoice PDF rendering",
  "log": ["1/5 — Pulling latest master...", "2/5 — Publishing (dotnet publish)...", "..."],
  "started_at": "2026-08-06T22:05:24Z",
  "finished_at": "2026-08-06T22:09:01Z"
}
```

---

## Networking / access

- Reachable only via Caddy (`stella-deployment-api.foxcraft.digital`, port 443) — no other port exposed. Inherits the same IP restriction as every other Caddy-routed service.
- ddashboard's outbound IP is already one of the two whitelisted static IPs, so no additional firewall work was needed for ddashboard specifically to reach this.
- SSH-signature auth is a second, independent layer on top of the IP restriction — even a request from an authorized IP is rejected without a valid signature from the `ddashboard` principal's key.

## Caddy block
```caddyfile
stella-deployment-api.foxcraft.digital {
    reverse_proxy deploy-api:8080
}
```

**Gotcha hit during setup:** after editing the Caddyfile, `docker compose up -d caddy` alone did **not** pick up the change — Compose only restarts a container if it detects a change to the service's own definition (image, env, etc.), not a bind-mounted file's content. Use `docker compose restart caddy` after any Caddyfile edit.

## DNS / cert issuance note

First cert request for this subdomain failed once on Let's Encrypt with `NXDOMAIN` during "secondary validation" — one of Let's Encrypt's multiple DNS vantage points hit a stale negative cache for the brand-new `_acme-challenge` TXT record, even though the record was already correctly live (confirmed via an independent external DNS checker). **Caddy's built-in automatic fallback to ZeroSSL** retried and succeeded about 40 seconds later without manual intervention. Expected/normal for a freshly created subdomain — let the full retry cycle run rather than assuming a first failure means something is broken.

---

## End-to-end test (confirmed working 2026-08-06)

From ddashboard, using its existing `~/.ssh/id_ed25519`:
1. Signed and POSTed a deploy request → received a `job_id`, confirming signature verification passed
2. Initial `git pull` failed (`Host key verification failed`) — fixed by mounting `/root/.ssh` into the container
3. Re-ran → `git pull`, `dotnet publish`, zip, and Azure deploy all progressed correctly through the SSE stream in real time, ending in a successful Azure deployment
