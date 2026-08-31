# Infrastructure

## Servers

### Stella (Hetzner Dedicated)
- **OS:** Ubuntu 24.04, kernel `6.8.0-137-generic` (patched 2026-08-06; server had been running 4+ months unpatched/unrebooted prior — check `uname -r` vs `apt list --upgradable` periodically)
- **CPU:** Intel i7-8700
- **RAM:** 124 GB
- **WAN interface:** `eno1` — this matters for firewall rules, see Security section
- **Role:** AI inference, API services, IMAP sync, reverse-proxied dev/staging apps

### ddashboard (Hetzner Managed)
- **Role:** WordPress — ddashboard theme, Stella client, REST API routes

---

## Services on Stella

Core Docker-managed services live at `/opt/services/docker-compose.yml`. Dev/staging apps live under `/opt/apps/`, one folder per app, each with their own `docker-compose.yml` (see "App deployment pattern" below).

| Service | Runtime | Port | Notes |
|---------|---------|------|-------|
| `caddy` | Docker (custom build, see below) | 443 (public); 8080 published but no longer routed | Reverse proxy — `stella.foxcraft.digital` block serves `/ollama`, `/imap-sync`, `/stella`; subdomain-based blocks for dev/staging apps |
| `stella-api` | Docker (`network_mode: host`) | 8001 | FastAPI via uvicorn — chat-only (`/chat/health`, `/chat/stream`) |
| `imap-sync` | Docker (Node.js) | 3001 | `imapsync` wrapper (Express) |
| `deploy-api` | Docker (`edge` network, no host port) | — | SSH-signature-authenticated deploy trigger — [`deploy-api.md`](deploy-api.md) |
| `ollama` | Docker (`edge` network) | 11434 (published to `127.0.0.1` only) | LLM inference — containerized 2026-08-10, see below |
| `health-api` | Docker (`edge` network) | 443 via Caddy (`stella-health-api.foxcraft.digital`) | System status endpoint for Atlas — [`health-api.md`](health-api.md) |
| `backup` | Docker (`edge` network, no host port) | — | Nightly restic backup to Hetzner Storage Box — see [`backup.md`](backup.md) |
| `osgar-datahub-dev` | Docker (.NET dev, hot-reload) | 5081→8080 (localhost) | `.NET` app, same pattern as `advoapp-dev` — `Dockerfile.dev` + bind-mounted `src`; compose file at `/opt/apps/dotnet/osgar.datahub.foxcraft.digital/docker-compose.dev.yml` |
| `osgar-datahub-db` | Docker (MSSQL 2022) | — (internal only) | `mcr.microsoft.com/mssql/server:2022-latest`, data volume `osgar-datahub-mssql-data`; same compose file as `osgar-datahub-dev` |

### Non-Docker systemd services
- `fail2ban.service`
- `docker-user-firewall.service` — reapplies `DOCKER-USER` iptables rules after every boot (see Security)

**`ollama.service` retired 2026-08-10** — Ollama moved to a Docker container (see "Ollama" section below). The systemd unit is stopped and disabled, not deleted — kept as an emergency fallback (`systemctl start ollama`) but should not come back on its own after a reboot.

**Removed 2026-08-06:** `chromadb.service`, `chromadb` pip package, `github-runner` container (never successfully registered — 404 on GitHub's runner-registration endpoint, confirmed zero configured runners via GitHub UI, no dependent workflows).

---

## Caddy — rebuilt for automatic HTTPS via Hetzner DNS-01

Standard Caddy can't issue certs without exposing port 80 to the whole internet (HTTP-01 challenge). Since all dev/staging subdomains are IP-restricted, Caddy is built with the **`caddy-dns/hetzner/v2`** plugin instead, which proves domain ownership via a DNS TXT record (DNS-01) — no inbound port ever needs to be open to Let's Encrypt/ZeroSSL.

**⚠️ Hetzner DNS API migration (2026-08-06):** Hetzner shut down the old `dns.hetzner.com` DNS Console/API in May 2026 and merged DNS management into the unified Hetzner Cloud API (`api.hetzner.cloud`). Tokens created in the old DNS Console **do not work** with the new API. If you ever see a `301` redirect to `console.hetzner.com` from a `dns.hetzner.com` API call, this is why. Use:
- **`caddy-dns/hetzner/v2`** (not v1) — v1 targets the dead API
- API tokens from **console.hetzner.com** → project (`konsoleH`) → Security → API tokens — not the old DNS Console
- Endpoint: `https://api.hetzner.cloud/v1/zones/{zone_id}/rrsets` (not the old `/api/v1/records` structure)

```dockerfile
# /opt/services/caddy/Dockerfile
FROM caddy:builder AS builder
RUN xcaddy build --with github.com/caddy-dns/hetzner/v2

FROM caddy:alpine
COPY --from=builder /usr/bin/caddy /usr/bin/caddy
```

Token stored in `/opt/services/caddy/.env` (`chmod 600`), referenced in the Caddyfile as `{env.HETZNER_API_TOKEN}`. **Never run `curl -v` against Hetzner's API with the token in a header** — verbose curl output prints request headers including `Auth-API-Token`/`Authorization`. Use plain `curl -s` and only inspect the response body.

**foxcraft.digital zone ID:** `567656` (Hetzner Cloud API, `konsoleH` project)

### Caddyfile structure

**Updated 2026-08-07:** Ollama, imap-sync, and stella-api moved from a bare `:8080` block to a proper `stella.foxcraft.digital` subdomain with automatic HTTPS. The `:8080` block no longer exists in the Caddyfile; port 8080 is still published by the `caddy` container in `docker-compose.yml` but receives no traffic.

Each site block must pin Let's Encrypt **production** via a per-site `tls { issuer acme { ... } }` block that includes an explicit `dir` — see "Issue 3 / staging fallback" below for why a bare `issuer acme` (or a global `acme_dns` alone) is not enough.

```caddyfile
{
    email projects@foxcraft.digital
    acme_dns hetzner {env.HETZNER_API_TOKEN}
}

stella.foxcraft.digital {
    tls {
        issuer acme {
            email projects@foxcraft.digital
            dir https://acme-v02.api.letsencrypt.org/directory
            dns hetzner {env.HETZNER_API_TOKEN}
        }
    }
    handle /ollama* {
        uri strip_prefix /ollama
        reverse_proxy ollama:11434 {
            transport http {
                response_header_timeout 30m
                dial_timeout 30s
            }
        }
    }
    handle /imap-sync* {
        uri strip_prefix /imap-sync
        reverse_proxy imap-sync:3001
    }
    handle /stella* {
        uri strip_prefix /stella
        reverse_proxy 172.18.0.1:8001
    }
}

advoapp.finditoo.foxcraft.digital {
    tls {
        issuer acme {
            email projects@foxcraft.digital
            dir https://acme-v02.api.letsencrypt.org/directory
            dns hetzner {env.HETZNER_API_TOKEN}
        }
    }
    reverse_proxy advoapp-dev:8080
}

stella-deployment-api.foxcraft.digital {
    tls {
        issuer acme {
            email projects@foxcraft.digital
            dir https://acme-v02.api.letsencrypt.org/directory
            dns hetzner {env.HETZNER_API_TOKEN}
        }
    }
    reverse_proxy deploy-api:8080
}

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

osgar.datahub.foxcraft.digital {
    tls {
        issuer acme {
            email projects@foxcraft.digital
            dir https://acme-v02.api.letsencrypt.org/directory
            dns hetzner {env.HETZNER_API_TOKEN}
        }
    }
    reverse_proxy osgar.datahub.foxcraft.digital:8080
}
```

**Required template for every new domain block** (copy as-is; only change hostname + upstream):

```caddyfile
<subdomain>.foxcraft.digital {
    tls {
        issuer acme {
            email projects@foxcraft.digital
            dir https://acme-v02.api.letsencrypt.org/directory
            dns hetzner {env.HETZNER_API_TOKEN}
        }
    }
    reverse_proxy <container-name>:8080
}
```

The `dir` line is decisive — without it, Caddy can still fall back internally to `acme-staging-v02.api.letsencrypt.org` even with `issuer acme` set. A global `acme_dns hetzner {env.HETZNER_API_TOKEN}` alone also does **not** lock the issuer.

**Already pinned with `dir`:** `stella.foxcraft.digital`, `advoapp.finditoo.foxcraft.digital`, `stella-deployment-api.foxcraft.digital`, `stella-health-api.foxcraft.digital`, `osgar.datahub.foxcraft.digital`.

Each new subdomain gets its own block at the bottom, reverse-proxying to a container name on the `edge` network — Caddy handles TLS automatically via DNS-01, no manual cert management needed.

**After any Caddyfile edit:** `docker compose restart caddy` is required (not `up -d` alone — Compose does not recreate/reload on bind-mounted file content changes). Restart mid-retry is fine; Caddy simply restarts the attempt for the affected domain.

**Resolved 2026-08-10:** Ollama's Caddy route now uses `ollama:11434` container-name routing, consistent with `imap-sync:3001` and `deploy-api:8080`. `stella-api` still uses `172.18.0.1:8001` (the `services_default` gateway) since it runs `network_mode: host` and isn't reachable by container name — this one is a structural constraint of that networking mode, not an inconsistency to fix.

**Gotcha 2026-08-17:** The live Caddyfile still had `172.18.0.1:11434` for the Ollama route (the old bridge-gateway address) despite the doc saying `ollama:11434`. Ollama only listens on `127.0.0.1:11434` (not on bridge interfaces), so every `POST /ollama/api/chat` hit an i/o timeout and Caddy returned 502. Fixed by updating the Caddyfile to `ollama:11434` and adding an explicit transport block (`response_header_timeout 30m`, `dial_timeout 30s`) to handle slow model cold-starts.

**New gotcha found while building `health-api`:** any *new* container that needs to reach `stella-api` must use the gateway IP of **its own** Docker network, not necessarily `172.18.0.1`. `health-api` (on `edge`) needed `172.20.0.1` — `edge`'s own gateway — not `services_default`'s. Using the wrong gateway doesn't error, it just times out silently, easily mistaken for a firewall block. Additionally, UFW must explicitly whitelist whatever subnet the calling container's network uses (see Security section below) — this was missed for `edge` on port `8001` until `health-api` surfaced it.

---

### Caddy / cert issuance operational notes

Three issues hit during the 2026-08-07 `stella.foxcraft.digital` migration — worth knowing on sight to avoid repeating the debug cycle.

#### Issue 1 — `env_file` changes don't take effect with `docker compose restart`

`docker compose restart <service>` restarts the existing container process; it does **not** re-read `env_file`. Compose only re-evaluates `env_file` when the container is actually recreated.

**Fix:** after any `.env` change for a Compose service:
```bash
docker compose up -d --force-recreate <service>
# or for services with a build context:
docker compose down <service> && docker compose up -d --build <service>
```
This applies to any service using `env_file:` — not just Caddy. Verify the env is live inside the container with `docker exec <container> printenv <VAR>`.

#### Issue 2 — Stale `_acme-challenge` TXT records cause DNS-01 failures across retry attempts

Caddy deletes the `_acme-challenge` TXT record after an attempt **only when that attempt fully succeeds**. On every failure (NXDOMAIN timing, SERVFAIL, wrong issuer, interrupt), the record with the **old** token stays. The next attempt generates a **new** token — Let's Encrypt then either sees the old value (`Incorrect TXT record ... found`) or writing the new value fails at the Hetzner API with `duplicate value`.

Failure patterns:

- `NXDOMAIN looking up TXT for _acme-challenge.<name>` — validator queried before TXT record propagated (normal on first attempt, resolves on retry).
- `Incorrect TXT record "<value>" found` — validator found a **stale** TXT value from a previous failed/interrupted attempt.
- Hetzner `duplicate value` — Caddy tried to write a new token while the old one still exists under the same name.

**Root cause:** any incomplete attempt leaves the TXT behind — not only a manual interrupt (`docker compose restart` / `down` / `--force-recreate` mid-attempt), but also natural ACME failures (timing, wrong CA).

**Diagnose** (look for leftover challenge records for the subdomain):
```bash
source /opt/services/caddy/.env
curl -s -X GET "https://api.hetzner.cloud/v1/zones/567656/rrsets" \
  -H "Authorization: Bearer $HETZNER_API_TOKEN" | python3 -m json.tool | grep -B2 -A6 "_acme-challenge.<subdomain>"
```

**Fix — delete the record, then restart Caddy:**
```bash
source /opt/services/caddy/.env
curl -s -X DELETE \
  "https://api.hetzner.cloud/v1/zones/567656/rrsets/_acme-challenge.<subdomain>/TXT" \
  -H "Authorization: Bearer $HETZNER_API_TOKEN"

cd /opt/services
docker compose restart caddy
docker compose logs -f --since 1m caddy | grep -i "<subdomain>"
```

If DELETE returns `"not_found"`: fine — Caddy already cleaned up on the last successful (or partially cleaned) attempt.

Optional: confirm the record is actually live on the authoritative nameserver (bypasses resolver cache):
```bash
dig @ns1.your-server.de _acme-challenge.<subdomain> TXT +short
```

A clean first attempt typically resolves in 15–30 s once DNS has propagated; first-ever issuance for a new subdomain may take a few minutes across a few retries. Prefer letting an uninterrupted backoff cycle run; only delete + restart when stuck on stale/duplicate TXT errors.

**What was ruled out:** concurrent issuance across multiple domains (tested `stella.foxcraft.digital` in isolation — same NXDOMAIN pattern, so it's not a cross-domain race); UFW / IP whitelist (DNS-01 validation is purely outbound from Caddy to Hetzner's API, no inbound traffic involved).

#### Issue 3 — ZeroSSL / Let's Encrypt staging fallback (must pin `dir`)

**Problem:** A Caddyfile block without an explicit `tls { issuer acme { ... dir ... } }` pin can, after failed attempts, automatically fall back to ZeroSSL or Let's Encrypt **staging**. Staging certificates are not browser-trusted and never become valid — the domain block stays effectively broken, without that being obvious on a first glance at the logs.

**Root cause:** A global `acme_dns hetzner {env.HETZNER_API_TOKEN}` alone does **not** lock the issuer. Even `tls { issuer acme { dns hetzner ... } }` **without** an explicit `dir` still allows Caddy to switch between production and staging endpoints.

Observed history:
- ZeroSSL fallback after a Let's Encrypt failure once hung with no log output for 5+ minutes (TXT was correctly live — not a propagation issue).
- After repeated fast failures, Caddy escalated to `acme-staging-v02.api.letsencrypt.org`. Staging may succeed (proving dns-01 works) while production stays broken if the block remains unpinned.

**Fix — every new domain block from day one** (see Caddyfile template above): include `dir https://acme-v02.api.letsencrypt.org/directory` inside `issuer acme { ... }`. That line is what prevents the internal staging fallback.

**Do not** treat `acme-staging-v02` in the logs as harmless rate-limit protection to ignore — if staging is in play, the site is at risk of serving an untrusted cert. Pin `dir`, clean any stale `_acme-challenge` TXT, restart Caddy, and confirm the final issuer is `acme-v02.api.letsencrypt.org-directory` (or inspect the served chain with `curl -v`).

**Follow-up (2026-08-10, `stella-health-api.foxcraft.digital`):** staging fallback was also observed **with a correct `dir` pin already present**, on Caddy's own ~60s internal retry after an initial NXDOMAIN — not a config-reload restart. Waiting it out was not enough; a full `docker compose restart caddy` after DNS had propagated landed on production LE. See [`health-api.md`](health-api.md). If this recurs: confirm DNS live, then force a Caddy restart rather than relying on internal retries alone.

---

## App deployment pattern — `edge` network

**Goal: adding a new dev/staging app should never require new firewall work.**

All new apps (dotnet dev environments, WP staging sites, etc.) join a shared Docker network called `edge` and publish **no host ports at all**. Caddy is the only thing with ports actually exposed on the host; it reaches every app internally by container name.

```bash
docker network create edge   # one-time setup, already done
```

### Folder convention

Folders are named by subdomain, for easy visual mapping:

```
/opt/apps/
  ├── dotnet/
  │     └── advoapp.finditoo.foxcraft.digital/
  │           ├── src/              ← git clone of the app repo
  │           ├── Dockerfile
  │           └── docker-compose.yml
  └── wp-staging/
        └── <subdomain>/
              └── docker-compose.yml
```

**Container names also match the subdomain** — makes `docker ps`/`docker logs` output unambiguous once there are many services running. Example:

```yaml
services:
  advoapp.finditoo.foxcraft.digital:
    build: .
    container_name: advoapp.finditoo.foxcraft.digital
    environment:
      - ASPNETCORE_URLS=http://+:8080
      - ASPNETCORE_ENVIRONMENT=Development
    networks:
      - edge
    restart: unless-stopped

networks:
  edge:
    external: true
```

### Deploying a new app — checklist

1. `mkdir -p /opt/apps/<category>/<subdomain>`
2. Clone source into `src/` (see Git/SSH access below)
3. Write a `Dockerfile` appropriate to the stack (see `finditoo-advoapp` for a .NET example)
4. Write `docker-compose.yml` per the template above — **no `ports:` block**
5. Create the DNS A record via Hetzner Cloud API (see below)
6. Add a Caddy block with explicit `dir` pin (see Caddyfile template above), then `docker compose restart caddy`
7. Check / clear stale `_acme-challenge` TXT if issuance sticks (Issue 2)
8. `docker compose build && docker compose up -d` (or `docker-compose.dev.yml` for .NET hot-reload — see [`dotnet-app-deployment.md`](dotnet-app-deployment.md))

No UFW or `DOCKER-USER` changes needed for any of this — that's the entire point of routing everything through Caddy on the `edge` network.

### Gateway IP depends on which Docker network the caller is on

Every Docker bridge network has its own gateway IP. A container on `edge` and a container on `services_default` will get different gateway addresses even though both ultimately reach the same host. When a container needs to reach a `network_mode: host` service (like `stella-api`) via its gateway IP, always check *that specific container's* network membership and gateway — don't assume a gateway IP that worked from one network will work from another:

```bash
docker inspect <container> --format '{{range $net, $conf := .NetworkSettings.Networks}}{{$net}}: {{$conf.Gateway}}{{"\n"}}{{end}}'
```

Combine this with a UFW check on the destination port — Docker network membership does not imply firewall access. Both must be correct, and a mismatch on either one produces the same symptom (a silent timeout), making them easy to conflate during debugging.

### Git/SSH access for cloning private repos

A general-purpose SSH key was generated on Stella (`~/.ssh/id_ed25519`, default path/name, no passphrase — needs to work unattended) and added to Dominik's **personal GitHub account** (not a repo-specific deploy key, not a machine user — deliberate choice for simplicity; consider migrating to a dedicated machine-user GitHub account later if broader/more permanent server-level access is wanted, since this key currently inherits access to everything the personal account can see).

```bash
git clone git@github.com:foxcraftdigital/finditoo-advoapp.git src
```

### Creating a DNS record via the Hetzner Cloud API

```bash
source /opt/services/caddy/.env
curl -s -X POST "https://api.hetzner.cloud/v1/zones/567656/rrsets" \
  -H "Authorization: Bearer $HETZNER_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "<subdomain-part>",
    "type": "A",
    "ttl": 300,
    "records": [{"value": "95.217.144.93"}]
  }'
```

**Known non-issue:** ProtonVPN's own DNS resolver (observed at `10.2.0.1`) is slow/unreliable picking up brand-new records, even though public resolvers (Google `8.8.8.8`, Cloudflare `1.1.1.1`) reflect them within seconds. This is not a security or server-side issue — just expect it every time a new subdomain is added, and don't waste time debugging the server over it. Workarounds: query a public resolver directly (`dig @8.8.8.8 ...`) to confirm the record is live, or use `curl --resolve <domain>:443:95.217.144.93 ...` to bypass DNS entirely for testing.

### Worked example: `advoapp.finditoo.foxcraft.digital`

.NET 10 app (`finditoo-advoapp`), no database, deployed 2026-08-06 following the exact pattern above. Verified working end-to-end: valid TLS cert (Let's Encrypt via DNS-01 — originally issued via ZeroSSL fallback on 2026-08-06, then re-issued against Let's Encrypt production on 2026-08-07 when per-site `tls { issuer acme }` pinning was applied), correct app response through Caddy, and — critically — verified as **actually IP-restricted** by testing from both a whitelisted IP (succeeds) and an unauthorized IP with VPN off (times out). See Security section for why that verification mattered.

---

## Ollama — containerized 2026-08-10

Moved from a native systemd service to a Docker container, for three reasons: consistency with every other service on the `edge` network pattern, proper CPU resource limiting (the systemd `CPUQuota`/`AllowedCPUs` settings documented in an earlier version of this doc were never actually applied — confirmed via `systemctl cat ollama.service` returning empty for both), and to fix the gateway-IP routing inconsistency flagged as an open follow-up (Caddy's `/ollama` route now uses `ollama:11434` container-name routing instead of the `172.18.0.1` bridge gateway workaround).

```yaml
# /opt/services/docker-compose.yml (relevant service)
ollama:
  image: ollama/ollama:latest
  container_name: ollama
  volumes:
    - ollama-models:/root/.ollama
  environment:
    - OLLAMA_KEEP_ALIVE=30m
  ports:
    - "127.0.0.1:11434:11434"
  deploy:
    resources:
      limits:
        cpus: "6"
  cpuset: "0-5"
  networks:
    - edge
  restart: unless-stopped
```

**Resource limits:** capped at half the machine — `cpus: "6"` (Docker's `cpus` is in units of full cores, not threads; the i7-8700 has 6 cores / 12 threads, confirmed via `nproc`) and `cpuset: "0-5"` pins to the first 6 threads specifically. Both were previously intended at the systemd level but never actually enforced — this is the first time a real limit has existed.

**`OLLAMA_KEEP_ALIVE=30m`** — a loaded model stays resident for 30 minutes after its last use, then unloads automatically. Deliberately *not* set to `-1` (unload-never) despite that being tried first: an always-resident 42GB+ model regardless of actual use was judged not worth permanently holding that much RAM hostage, especially now that Ollama shares the box with everything else under a hard 6-core CPU cap (see below) rather than running unconstrained as it did before containerization.

**No model loads automatically on container start** — this was true under both the old `-1` setting and the current `30m` one. Whatever's needed gets loaded on first request (or via a manual `ollama run <model> ""` warm-up) and then follows the 30-minute idle-unload policy from that point.

**Superseded "always-warm" plan:** an earlier version of this doc (and the original systemd-era setup) described `llama3.3:70b` and `phi4-mini` as "preloaded permanently." That framing no longer applies — both were explicitly unloaded (`ollama stop`) once `OLLAMA_KEEP_ALIVE` was changed to `30m`, and neither is treated as special/always-resident going forward. If a future need for a genuinely always-warm model reappears, the fix is model-specific `keep_alive` values in the actual API calls (Ollama supports per-request overrides), not a blanket container-wide `-1`.

**Port binding:** `127.0.0.1:11434:11434` — published only to localhost, since `stella-api` (which runs `network_mode: host`) needs to reach it via `127.0.0.1`. Everything else reaches it via `ollama:11434` container-name routing on `edge`.

**Models are not backed up.** The `ollama-models` Docker volume holds ~172GB of blobs, fully re-downloadable via `ollama pull`. Excluded from the restic backup scope by design — same principle as before containerization, just a different storage location. The volume also contains an auto-generated SSH keypair (`id_ed25519`/`.pub`) used for Ollama's own registry/cloud auth — not user-configured, regenerates automatically, not worth preserving.

**Migration note:** models were re-pulled fresh into the new container rather than migrating the old systemd-managed blob directory — simpler, and the recovery story (re-pull on demand) was already the plan either way. Old host-side model files can be safely deleted once `docker exec ollama ollama list` confirms all expected models are present in the container.

**UFW:** the old rules for port `11434` (whitelisted to the two static IPs) are no longer needed once the systemd service is confirmed fully retired, since nothing needs to reach Ollama directly from outside the Docker network anymore — remove via `ufw status numbered | grep 11434` then `ufw delete <rule number>`.

---

## ChromaDB — REMOVED 2026-08-06

Fully decommissioned. See git history / prior doc versions if a vector search feature is ever rebuilt — would need ChromaDB and `nomic-embed-text` reinstalled from scratch, plus new `stella-api` routes and a new WordPress-side client, none of which exist anymore. See [`../integration/email-indexing.md`](../integration/email-indexing.md) for the full decommission notice.

---

## Security

### Current state (verified 2026-08-06; ports updated 2026-08-07)

- **UFW** — active, default-deny incoming. Ports 22, 443 restricted to `194.126.177.181` and `23.88.90.12`. Port 8001 additionally allows `172.18.0.0/16` (`services_default` bridge subnet) for Caddy's internal proxy calls, **and `172.20.0.0/16` (`edge` bridge subnet, added 2026-08-10)** for `health-api`'s stella-api check. Port 11434's direct-access rules removed 2026-08-10 once Ollama moved fully behind the `edge` network / Caddy — no longer needs a host-level UFW allowance for the old static-IP whitelist.
- **fail2ban** — running
- **Docker-published ports (8080, 3001, 443)** — protected via a custom `DOCKER-USER` iptables chain (details below)

**Note:** port 8080 DOCKER-USER rules are now stale — the `:8080` Caddyfile block was removed on 2026-08-07 (traffic moved to `stella.foxcraft.digital:443`). Port 8080 is still published by the Caddy container but nothing routes to it. The DOCKER-USER rules blocking unauthorized access to 8080 are harmless to leave in place and serve as defense-in-depth; they can be removed in a future cleanup pass if desired.

### ⚠️ Docker-published ports bypass UFW entirely

Any port published via `docker-compose.yml` (`ports: ["X:X"]`) is handled by Docker's own iptables NAT/FORWARD rules, evaluated **before** UFW's INPUT chain. Plain `ufw allow`/`deny` on these ports has **zero effect**, regardless of what `ufw status` reports. This applies to `caddy` (8080, 443) and `imap-sync` (3001).

### The DOCKER-USER fix — three iterations, document all of them

This took three attempts to get right, and the failure modes are worth remembering since they're non-obvious:

**Attempt 1 — wrong rule order.** Used `iptables -I DOCKER-USER 1` (insert-at-top) for every rule including DROP, in a script where DROP was written last. Since each `-I 1` pushes to the very top, the DROP rules ended up **above** the ACCEPT rules — blocking everyone, including whitelisted IPs. Fix: use `-A` (append) instead of `-I`, after an `iptables -F DOCKER-USER` flush, so rules land in the order written (ACCEPT rules first, DROP last).

**Attempt 2 — destination-IP matching doesn't work for inbound Docker traffic.** After fixing the order, outbound container traffic broke — `dotnet restore` timed out trying to reach `api.nuget.org:443`, because the DROP rule matched `tcp dpt:443` with no destination filter, catching outbound traffic too. The fix at the time added `-d $STELLA_IP` (Stella's own public IP) to scope the rules to inbound-looking traffic only. **This looked correct but was actually broken**: Docker DNATs (rewrites) the destination address of inbound connections to the container's internal bridge IP *before* the packet reaches `DOCKER-USER` — so a rule matching `-d 95.217.144.93` can **never match real inbound traffic** at all. It silently fell through to Docker's own default-accept rule. This is why a manual test from an external IP with VPN off succeeded even though the DROP rule's packet counter stayed at zero — the rule was structurally unable to ever fire.

**Attempt 3 — match on WAN interface instead (current, correct approach).** Use `-i eno1` (Stella's physical WAN NIC) to distinguish "packet arriving from the internet" from "packet leaving a container toward the internet," rather than trying to match on IP addresses that get rewritten mid-flight:

```bash
#!/bin/bash
# /usr/local/bin/docker-user-firewall.sh
WAN_IF="eno1"

iptables -F DOCKER-USER

iptables -A DOCKER-USER -i $WAN_IF -s 194.126.177.181 -p tcp --dport 3001 -j ACCEPT
iptables -A DOCKER-USER -i $WAN_IF -s 23.88.90.12 -p tcp --dport 3001 -j ACCEPT
iptables -A DOCKER-USER -i $WAN_IF -p tcp --dport 3001 -j DROP

iptables -A DOCKER-USER -i $WAN_IF -s 194.126.177.181 -p tcp --dport 8080 -j ACCEPT
iptables -A DOCKER-USER -i $WAN_IF -s 23.88.90.12 -p tcp --dport 8080 -j ACCEPT
iptables -A DOCKER-USER -i $WAN_IF -p tcp --dport 8080 -j DROP

iptables -A DOCKER-USER -i $WAN_IF -s 194.126.177.181 -p tcp --dport 443 -j ACCEPT
iptables -A DOCKER-USER -i $WAN_IF -s 23.88.90.12 -p tcp --dport 443 -j ACCEPT
iptables -A DOCKER-USER -i $WAN_IF -p tcp --dport 443 -j DROP
```

- Applied by: `docker-user-firewall.service` (systemd oneshot, `After=docker.service`, `WantedBy=multi-user.target`, enabled — survives reboot)
- **Verified working 2026-08-06** in both directions:
  - Unauthorized external IP (real ISP IP, VPN off) → `curl` times out after 75s
  - Whitelisted IP (via ProtonVPN) → connects normally, valid cert, correct response
  - Outbound container traffic (NuGet restore) → unaffected

**Lesson for any future firewall rule involving Docker-published ports:** never trust "the packet counter didn't increment" as proof a rule is working — a structurally-unreachable rule will show zero matches forever, which looks identical to "nothing tried to violate it." Always test from a genuine external client with a known non-whitelisted IP, not from the server testing against itself (self-directed traffic often routes differently, e.g. hairpin NAT, and doesn't cross `-i eno1` the same way real external traffic does).

### Don't install `iptables-persistent` alongside `ufw`

On this Ubuntu 24.04 setup, `apt install iptables-persistent` silently **removed the `ufw` package** as a conflicting dependency (both manage iptables state) — this briefly left the server's UFW rules completely unenforced without any obvious error. If `DOCKER-USER` rules ever need to survive a reboot without the systemd-service approach above, do **not** reach for `iptables-persistent`; it's not compatible with a ufw-managed host.

### 5678 (n8n) — removed

Stale UFW rule with no corresponding service/container/binary anywhere on the server. Removed 2026-08-06.

### Secrets handling

- **Never use `curl -v`/`curl --verbose` with an API token in a header** — verbose mode prints the full request including headers. Use `curl -s` (silent) and inspect only the response body. This mistake happened once with a Hetzner API token during this session; the token was rotated afterward.
- API tokens live in `.env` files with `chmod 600`, scoped as narrowly as the provider allows (e.g. Hetzner token scoped to the `konsoleH` project specifically, not account-wide).

---

## Docker Compose (core services)

Location: `/opt/services/docker-compose.yml`

```yaml
services:
  caddy:
    build: ./caddy
    env_file:
      - ./caddy/.env
    ports:
      - "8080:8080"
      - "443:443"
    volumes:
      - ./caddy/Caddyfile:/etc/caddy/Caddyfile
    extra_hosts:
      - "host.docker.internal:host-gateway"
    networks:
      - default
      - edge
    restart: unless-stopped

  imap-sync:
    build: ./imap-sync
    restart: unless-stopped
    ports:
      - "3001:3001"

  stella-api:
    build: ./stella-api
    network_mode: host
    environment:
      - OLLAMA_URL=http://127.0.0.1:11434
    restart: unless-stopped

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

  ollama:
    image: ollama/ollama:latest
    container_name: ollama
    volumes:
      - ollama-models:/root/.ollama
    environment:
      - OLLAMA_KEEP_ALIVE=30m
    ports:
      - "127.0.0.1:11434:11434"
    deploy:
      resources:
        limits:
          cpus: "6"
    cpuset: "0-5"
    networks:
      - edge
    restart: unless-stopped

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

networks:
  edge:
    external: true

volumes:
  ollama-models:
```
