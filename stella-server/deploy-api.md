# Deploy API — SSH-Signature-Authenticated Deploy Trigger

**Purpose:** lets ddashboard (or any other trusted caller) remotely trigger a production deploy (currently just `advoapp-production`) over HTTP, with real-time status, without a shared-secret bearer token.

**Live at:** `https://stella-deployment-api.foxcraft.digital`

**Why SSH-signature auth instead of a bearer token/API key:** reuses SSH's existing trust model. ddashboard already has SSH keys; instead of minting and managing a separate API secret (that would need rotation, secure storage, risk of leaking in logs), the caller **signs a short-lived payload** with its own SSH private key using OpenSSH's native `ssh-keygen -Y sign`/`-Y verify` (the same mechanism used for signed git commits) — the private key itself never leaves ddashboard, never crosses the network.

---

## Architecture

Containerized, on the `edge` Docker network (no published host port — same pattern as every other Stella service). Runs as a FastAPI app under Uvicorn.

**Why containerized rather than a native systemd service (as first attempted):** a systemd-native version was built and then abandoned mid-setup once it became clear it broke the "everything routes through Caddy on `edge`, zero new firewall work per service" principle established earlier in the day — it would have needed its own bespoke UFW rule. Containerizing it, with the Docker socket mounted in so it can still launch sibling build containers, keeps it consistent with the rest of the stack.

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

**`/root/.ssh:/root/.ssh:ro` is essential** — the container needs the same SSH key + `known_hosts` Stella's host already uses to clone/pull the private GitHub repo. Without this mount, the deploy script's `git pull` step fails inside the container with `Host key verification failed` / `Could not read from remote repository`, even though the exact same script works fine when run directly on the host (which already has SSH trust established). This was hit and fixed during initial setup — worth remembering for any future containerized service that needs git access.

**Dockerfile** (`/opt/services/deploy-api/Dockerfile`) installs: `git`, `zip`, `openssh-client` (needed for `ssh-keygen -Y verify`), Docker CLI (to launch sibling containers via the mounted socket), and the Azure CLI (so the mounted `deploy-advoapp.sh` script can call `az` from inside this container).

---

## Auth mechanism

**Allowed signers file** (`/opt/services/deploy-api/allowed_signers`):
```
ddashboard ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIOpVG5e5AzGVVKU9uIWEACauLbP/w+SOTw3MnSPqruVn foxcraft-digital@dedi6724
```
Format: `<principal> <key-type> <base64-key>`. `ddashboard` is the "principal" name checked against in the API's verification call — arbitrary, just needs to match on both sides.

**Payload format:** `deploy-<app_name>:<unix_timestamp>`, e.g. `deploy-advoapp-production:1786045000`. The timestamp provides replay protection — the API rejects any payload more than 60 seconds old or in the future (`REPLAY_WINDOW_SECONDS` in `main.py`).

**Signing (caller side, e.g. on ddashboard):**
```bash
TIMESTAMP=$(date +%s)
PAYLOAD="deploy-advoapp-production:${TIMESTAMP}"
echo -n "$PAYLOAD" > /tmp/payload.txt
ssh-keygen -Y sign -f ~/.ssh/id_ed25519 -n deploy /tmp/payload.txt
```
Produces `/tmp/payload.txt.sig` — an armored text signature. `-n deploy` sets the "namespace," which must match the API's `NAMESPACE = "deploy"` constant on both sides.

**Verification (server side, inside the API):**
```python
subprocess.run([
    "ssh-keygen", "-Y", "verify",
    "-f", ALLOWED_SIGNERS_PATH,
    "-I", PRINCIPAL,      # "ddashboard"
    "-n", NAMESPACE,      # "deploy"
    "-s", sig_path,
], input=payload, ...)
```
Exit code 0 = valid signature from a key listed in `allowed_signers` for that principal/namespace.

**Sending the request:**
```bash
curl -s -X POST https://stella-deployment-api.foxcraft.digital/deploy/advoapp-production \
  -H "Content-Type: application/json" \
  -d "$(python3 -c "
import json
payload = open('/tmp/payload.txt').read()
sig = open('/tmp/payload.txt.sig').read()
print(json.dumps({'payload': payload, 'signature': sig}))
")"
```

---

## Endpoints

| Method | Path | Purpose |
|---|---|---|
| GET | `/health` | Health check, no auth |
| POST | `/deploy/{app_name}` | Trigger a deploy. Body: `{"payload": "...", "signature": "..."}`. Returns `{"job_id": "...", "status": "queued"}` |
| GET | `/deploy/{app_name}/status/{job_id}` | Poll current job state (status, current step, full log so far) |
| GET | `/deploy/{app_name}/status/{job_id}/stream` | Server-Sent Events — real-time log tail as the deploy runs, ends with a `done` event |
| GET | `/deploy/{app_name}/latest` | Convenience: most recent job for that app, without needing to track `job_id` |

**App allowlist** (`APPS` dict in `main.py`) — only apps explicitly listed can be triggered; `app_name` is never used to construct an arbitrary filesystem path. Currently:
```python
APPS = {
    "advoapp-production": {
        "script": "/opt/services/azure-deploy/deploy-advoapp.sh",
    }
}
```
Adding a future app = one new entry here, pointing at its own deploy script. No other code changes needed.

**Job tracking:** in-memory only (a Python dict, guarded by a lock) — jobs are lost if the container restarts. Fine for this use case (deploy history isn't precious, and the `deploy.log` file persists a plain-text audit trail separately via `LOG_PATH`); would need a real datastore if job history needed to survive restarts or scale beyond one container replica.

---

## Networking / access

- Reachable only via Caddy (`stella-deployment-api.foxcraft.digital`, port 443) — no other port exposed. Inherits the same IP restriction as every other Caddy-routed service (currently the two whitelisted static IPs).
- ddashboard's outbound IP is already one of those two whitelisted IPs, so no additional firewall work was needed for ddashboard specifically to reach this.
- SSH-signature auth is a second, independent layer on top of the IP restriction — even a request from an authorized IP is rejected without a valid signature from the `ddashboard` principal's key.

## Caddy block
```caddyfile
stella-deployment-api.foxcraft.digital {
    reverse_proxy deploy-api:8080
}
```

**Gotcha hit during setup:** after editing the Caddyfile, `docker compose up -d caddy` alone did **not** pick up the change — Compose only restarts a container if it detects a change to the service's own definition (image, env, etc.), not a bind-mounted file's content. Use `docker compose restart caddy` (or `up -d --force-recreate`) after any Caddyfile edit to guarantee a reload.

## DNS / cert issuance note

First cert request for this subdomain failed once on Let's Encrypt with `NXDOMAIN` during "secondary validation" — one of Let's Encrypt's multiple DNS vantage points hit a stale negative cache for the brand-new `_acme-challenge` TXT record, even though the record was already correctly live (confirmed via an independent external DNS checker). **Caddy's built-in automatic fallback to ZeroSSL** retried and succeeded about 40 seconds later without any manual intervention — this is expected/normal behavior for a freshly created subdomain, not a sign of misconfiguration. Let the full retry cycle run rather than assuming a first failure means something is broken.

---

## End-to-end test (confirmed working 2026-08-06)

From ddashboard, using its existing `~/.ssh/id_ed25519`:
1. Signed and POSTed a deploy request → received a `job_id`, confirming signature verification passed
2. Polled/streamed status → `git pull` initially failed (`Host key verification failed`) — fixed by mounting `/root/.ssh` into the container (see Architecture above)
3. Re-ran after the fix → `git pull`, `dotnet publish`, zip, and Azure deploy all progressed correctly through the SSE stream in real time
