# Health API — System Status Endpoint for Atlas

**Purpose:** gives Atlas (or any other trusted caller) a single signed endpoint to poll for the live status of every container, HTTP service, TLS cert, and disk on Stella — closes the gap exposed on 2026-08-10 when `advoapp-dev` silently died and stayed down for 3 days with nothing monitoring it.

**Live at:** `https://stella-health-api.foxcraft.digital` — given its own subdomain 2026-08-10, same day as initial build, once it became clear Atlas (running on a separate host, `dev.atlas.foxcraft.digital`) can't reach an internal Docker network IP on Stella directly. Routed through Caddy like every other public-facing service; still no host port published by the container itself — Caddy is the only thing with a route to it.

**`STELLA_HEALTH_API_URL=https://stella-health-api.foxcraft.digital`**

---

## Verification status

**Fully confirmed, tested directly:**
- `GET /health` — basic liveness, tested both internally (container IP) and via the public route. Returns `{"status":"ok"}`.
- `POST /system/health` — full signed request/response cycle, tested via internal container IP with a temporary test principal (`stella-local`, since removed). Returned complete, correct JSON covering all checks active at that point: 8 container statuses, 2 HTTP checks, 4 TLS certs, disk, Ollama model count.
- **Auth correctly rejects invalid signatures** — confirmed when a request signed with Stella's own untrusted key (`stella-server`, not in `allowed_signers`) was correctly rejected with `401 signature verification failed`. This is the production-correct behavior: only `ddashboard` (Atlas's principal) should ever pass.
- Public subdomain `stella-health-api.foxcraft.digital` — live, valid Let's Encrypt production cert (not staging), reachable.

**Not yet re-verified through a full signed round-trip:**
- `check_docker_daemon()` and `check_ollama_resource_usage()` — added after the last full signed-response test (which used the temporary `stella-local` key, since removed to restore production-correct auth). Both are simple, isolated functions using the same subprocess pattern already proven working elsewhere in `health.py`, and the container rebuilt cleanly with no errors — but they haven't been observed returning real data through an actual end-to-end request.

**Fastest way to close this gap without reopening the temporary-key dance again** — call the two new functions directly inside the container, bypassing auth entirely (auth isn't what's in question, just the check logic itself):
```bash
docker exec health-api python3 -c "from health import check_docker_daemon, check_ollama_resource_usage; import json; print(json.dumps(check_docker_daemon())); print(json.dumps(check_ollama_resource_usage()))"
```

Once run, record the result here:
```
docker_daemon: <paste result>
ollama_resource_usage: <paste result>
```

---

## Why a separate service from `deploy-api`

Originally built as a route bolted onto `deploy-api` (`POST /system/health`), then split out deliberately:

- `deploy-api` already holds the highest-privilege access on the server — mounted Docker socket, mounted SSH key, Azure service principal credentials. Bundling health-checks into it means the monitoring system shares fate with the most sensitive service on the box: if `deploy-api` crashes or is mid-rebuild, the dashboard goes dark at exactly the moment visibility matters most.
- `health-api` only needs the Docker socket (read access to container state) and a read-only mount of the host root (disk stats) — genuinely lower privilege. No SSH key, no cloud credentials, no `/opt/apps` write access.
- A bug in one no longer takes down the other — proven directly during build/test, when `deploy-api` briefly broke from an unrelated edit and `health-api` was unaffected once split apart.

---

## Architecture

```
/opt/services/health-api/
├── Dockerfile
├── main.py       # FastAPI app, SSH-signature auth (same mechanism as deploy-api)
└── health.py      # all check logic
```

```yaml
# /opt/services/docker-compose.yml (relevant service)
health-api:
  build: ./health-api
  container_name: health-api
  volumes:
    - /var/run/docker.sock:/var/run/docker.sock
    - ./deploy-api/allowed_signers:/opt/services/health-api/allowed_signers:ro
    - /:/hostroot:ro
  networks:
    - edge
  restart: unless-stopped
```

**Reuses `deploy-api`'s `allowed_signers` file** (mounted read-only) rather than maintaining a second copy — same `ddashboard`/Atlas principal is already trusted there, and it's confirmed that ddashboard and Atlas run on the same host and share the same SSH keypair, so no new key setup was needed to bring Atlas onto this endpoint.

**Dockerfile requires `docker-ce-cli` specifically, not `docker.io`.** The `docker.io` apt package only ships `dockerd` and support binaries — not the `docker` CLI client. Install via Docker's official apt repo the same way `deploy-api` does:

```dockerfile
FROM python:3.12-slim

RUN apt-get update && apt-get install -y --no-install-recommends \
    curl ca-certificates gnupg openssh-client \
    && install -m 0755 -d /etc/apt/keyrings \
    && curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc \
    && chmod a+r /etc/apt/keyrings/docker.asc \
    && echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian $(. /etc/os-release && echo $VERSION_CODENAME) stable" > /etc/apt/sources.list.d/docker.list \
    && apt-get update \
    && apt-get install -y --no-install-recommends docker-ce-cli \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app
RUN pip install --no-cache-dir fastapi uvicorn
COPY main.py health.py .
EXPOSE 8080
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8080"]
```

### Public Caddy route

```caddyfile
stella-health-api.foxcraft.digital {
    tls {
        issuer acme {
            email projects@foxcraft.digital
            dir https://acme-v02.api.letsencrypt.org/directory
            dns hetzner {env.HETZNER_API_TOKEN}
        }
    }
    reverse_proxy health-api:8080
}
```

**First cert issuance for this subdomain hit two things worth knowing:**

1. **Initial NXDOMAIN on the `_acme-challenge` TXT lookup** — expected, per Issue 2 in `infrastructure.md`: the A record was created only moments before Caddy attempted validation, and Let's Encrypt's validator hit propagation delay.

2. **Caddy fell back to the staging CA on its automatic retry, despite the `dir` pin being correctly present in the Caddyfile the whole time.** This is new — previously, "no `dir` pin" was the only known cause of staging fallback (Issue 3). Here the pin was verified present via `awk`/`grep` before and after, ruling that out. The retry that fell to staging happened ~60 seconds after the first failure, on Caddy's own internal retry cycle — not a config-reload restart. **A full `docker compose restart caddy` (not just waiting out Caddy's internal retry) after confirming DNS had fully propagated resolved it on the next attempt, landing correctly on `acme-v02.api.letsencrypt.org`.** Cause not fully confirmed — possibly stale internal retry state that a full restart clears but an internal retry doesn't. If this recurs on a future subdomain, don't just wait it out — force a restart once DNS is confirmed live.

**No new UFW rules for the public subdomain.** `stella-health-api.foxcraft.digital` routes through Caddy on port 443, which is already whitelisted for the two static IPs — no new UFW rule was needed for the public route itself. The earlier UFW fix (`172.20.0.0/16` on port 8001) was a separate, internal concern — `health-api` reaching `stella-api` directly — unrelated to Atlas's external access path through Caddy.

---

## Auth — identical mechanism to `deploy-api`

Same SSH-signature scheme (`ssh-keygen -Y sign`/`-Y verify`, namespace `deploy`), same `ddashboard` principal. See [`deploy-api.md`](deploy-api.md#auth-mechanism) for the full explanation of why this pattern was chosen over a bearer token.

**Endpoint:**

| Method | Path | Auth | Purpose |
|---|---|---|---|
| GET | `/health` | none | Basic liveness check |
| POST | `/system/health` | signed (`system-health`) | Full status report |

**Signing a request** (identical pattern to triggering a deploy, just a different action string):
```bash
TIMESTAMP=$(date +%s)
PAYLOAD="system-health:${TIMESTAMP}"
echo -n "$PAYLOAD" > /tmp/payload.txt
ssh-keygen -Y sign -f ~/.ssh/id_ed25519 -n deploy /tmp/payload.txt
```

**Sending it** (from Atlas / any external trusted host):
```bash
curl -s -X POST https://stella-health-api.foxcraft.digital/system/health \
  -H "Content-Type: application/json" \
  -d "$(python3 -c "
import json
payload = open('/tmp/payload.txt').read()
sig = open('/tmp/payload.txt.sig').read()
print(json.dumps({'payload': payload, 'signature': sig}))
")"
```

Internal edge-network callers can still use `http://health-api:8080/system/health`.

**Response can take longer than a typical API call** — the full check suite (Docker daemon, 8 container inspects, 2 HTTP probes, TLS handshakes for each cert domain including this subdomain, `docker exec ollama ollama list`, `docker stats`, host CPU/RAM/disk) can run 10-20+ seconds depending on load, especially if Ollama is busy. Set client timeouts to at least 60s, not the usual 5-15s default.

---

## What it checks

| Category | How | Notes |
|---|---|---|
| **Docker daemon** | `docker info` | Reported separately from container checks — if the daemon itself is down/restarting (the actual root cause of the Aug 7 `advoapp-dev` outage), this shows the real cause distinctly instead of all 8 container checks failing with the same generic "missing" error and no indication of *why*. |
| **Container status** | `docker inspect <name>` via mounted socket | Status, `StartedAt`, `RestartCount` for all 8 core/app containers |
| **HTTP health** | `stella-api` and `ollama` endpoints | See addressing notes below — neither is reachable via naive `127.0.0.1` |
| **TLS cert expiry** | Raw `ssl`/`socket`, connects to `caddy:443` with SNI for each domain | Domains include `stella.foxcraft.digital`, `advoapp.finditoo.foxcraft.digital`, `stella-deployment-api.foxcraft.digital`, `osgar.datahub.foxcraft.digital`, and **`stella-health-api.foxcraft.digital`** (self). See hairpin NAT note below — connecting to the public domain/IP directly times out |
| **Ollama model count** | `docker exec ollama ollama list`, compares against `EXPECTED_OLLAMA_MODELS = 13` | Flags `incomplete` if fewer than expected — catches a partial/broken pull |
| **Ollama resource usage** | `docker stats ollama --no-stream` | Live CPU%/memory usage for the Ollama container specifically — the actionable number given the CPU cap added the same day (`cpus: "6"`). Host-wide load average (below) doesn't tell you whether Ollama itself is the thing near its ceiling. |
| **Host CPU load / RAM** | Reads `/proc/loadavg` and `/proc/meminfo` through the `/hostroot` mount already used for disk stats | `load_1m` near `12.0` on this 12-thread box means fully saturated. No new mount required — same `/:/hostroot:ro` volume already in place. |
| **Disk usage** | `os.statvfs` on `/hostroot` (mounted read-only host root) | Total/free/used% |

### Addressing gotchas hit during build — know these before adding new checks

1. **`127.0.0.1` inside a container means the container itself, not the host or a sibling container.** `check_http` originally pointed at `http://127.0.0.1:8001/...` for `stella-api` — always failed. Fixed:
   - `ollama` is on the same `edge` network → reachable by container name: `http://ollama:11434/...`
   - `stella-api` runs with `network_mode: host` → not reachable by container name at all. Needs the **`edge` network's own gateway IP**, not `services_default`'s. These are two different bridge networks with different gateways — confirmed `edge`'s gateway is `172.20.0.1` via `docker inspect health-api --format '{{range $net, $conf := .NetworkSettings.Networks}}{{$net}}: {{$conf.Gateway}}{{"\n"}}{{end}}'`. Using the wrong gateway (`172.18.0.1`, `services_default`'s) doesn't error — it just times out, which looks identical to a firewall block. Always verify which network the *calling* container is actually on before assuming a gateway IP.

2. **UFW must explicitly allow the `edge` subnet on any port `health-api` needs to reach.** Port `8001` (stella-api) was only whitelisted for `172.18.0.0/16` (`services_default`) — traffic from `edge` (`172.20.0.0/16`) was silently dropped, again indistinguishable from a timeout on the client side. Fixed with:
   ```bash
   ufw allow from 172.20.0.0/16 to any port 8001
   ```
   **Any future container added to `edge` that needs to reach a host-networked service on a UFW-restricted port will hit this same wall.** Check `ufw status numbered | grep <port>` and add the `edge` subnet explicitly rather than assuming Docker network membership implies firewall access — it doesn't.

3. **Checking a public domain's TLS cert from a container on the same host it's served from can hit a hairpin NAT wall.** Connecting directly to `stella.foxcraft.digital:443` (resolves to Stella's own public IP `95.217.144.93`) from inside `health-api` timed out — `curl -v` showed `Connection timed out after 5002 milliseconds`. This is the same class of self-directed-traffic quirk noted in the general infrastructure doc (see [`infrastructure.md`](infrastructure.md) — "Always test from a genuine external client... self-directed traffic often routes differently"). Fixed by connecting to the `caddy` container by name instead, using SNI to still request the correct domain's cert:
   ```python
   with socket.create_connection(("caddy", 443), timeout=5) as sock:
       with ctx.wrap_socket(sock, server_hostname=domain) as ssock:
           cert = ssock.getpeercert()
   ```
   This uses full default TLS verification (not `CERT_NONE` — that returns an empty cert dict since Python skips parsing when verification is disabled). Verification still succeeds because `server_hostname` (used for both SNI and hostname-checking) is the real domain, independent of which IP the socket actually connected to.

---

## Example response

```json
{
  "timestamp": "2026-08-10T22:38:09.747340+00:00",
  "docker_daemon": {"status": "up", "version": "29.7.2"},
  "containers": {
    "caddy": {"status": "up", "started_at": "2026-08-10T21:14:36Z", "restart_count": 0},
    "advoapp-dev": {"status": "up", "started_at": "2026-08-10T21:11:38Z", "restart_count": 0}
  },
  "http_checks": {
    "stella-api": {"status": "up", "code": 200},
    "ollama": {"status": "up", "code": 200}
  },
  "certs": {
    "stella.foxcraft.digital": {"expires": "Nov  5 15:56:46 2026 GMT", "days_left": 86, "status": "ok"},
    "stella-health-api.foxcraft.digital": {"expires": "Nov  8 12:00:00 2026 GMT", "days_left": 89, "status": "ok"}
  },
  "ollama_models": {"count": 13, "expected": 13, "status": "ok"},
  "ollama_resource_usage": {"cpu_pct": "412.30%", "mem_usage": "38.2GiB / 62GiB", "mem_pct": "61.61%"},
  "disk": {"total_gb": 1001.6, "free_gb": 645.7, "used_pct": 35.5},
  "cpu_ram": {"load_1m": 4.2, "load_5m": 3.8, "load_15m": 3.1, "ram_total_gb": 124.0, "ram_available_gb": 78.3, "ram_used_pct": 36.9}
}
```

`status` fields are one of `up`/`down`/`degraded`/`missing`/`error`/`ok`/`warning`/`incomplete` depending on the check — Atlas's rendering layer should treat anything other than `up`/`ok` as needing attention.

---

## What Atlas still needs to build

Stella-side endpoint is live. Atlas (`dev.atlas.foxcraft.digital`) should:

1. A scheduled job that signs a `system-health:<timestamp>` payload and POSTs it to `https://stella-health-api.foxcraft.digital/system/health` every few minutes (`STELLA_HEALTH_API_URL`).
2. Storage for the latest snapshot per server (or a small history) — a new model, or reuse of an existing pattern.
3. A status grid view — green/red per container and check, surfaced somewhere visible (dashboard home, or a dedicated health page).
4. Timeout tuned to at least 60s given the response-time note above.
5. Render the newer top-level keys (`docker_daemon`, `ollama_resource_usage`, `cpu_ram`) once they appear in the payload.

---

## Known limitations

- **No UFW status check.** Reading `ufw status` requires host network-namespace access a properly-isolated container doesn't have without `--privileged` or `--pid=host` — deliberately not granted, to keep this service's privilege minimal. If UFW status is ever wanted here, the clean approach is a small host-level cron job that writes `ufw status` output to a file, which this container then reads — not a live syscall from inside the container.
- **`EXPECTED_OLLAMA_MODELS = 13` is hardcoded.** Update this constant in `health.py` if the intended model list changes, or the check will falsely flag `incomplete` after a deliberate model removal.
- **`ollama_resource_usage` is specific to the `ollama` container only.** If CPU/memory checks are ever wanted for other containers, `check_ollama_resource_usage()` would need generalizing into a parameterized version (`check_container_resource_usage(name)`), same pattern as `check_container`.
