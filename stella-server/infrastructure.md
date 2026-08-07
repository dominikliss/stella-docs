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
| `ollama` | systemd | 11434 | LLM inference |

### Non-Docker systemd services
- `ollama.service`
- `fail2ban.service`
- `docker-user-firewall.service` — reapplies `DOCKER-USER` iptables rules after every boot (see Security)

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

Each site block now pins explicitly to Let's Encrypt only via a per-site `tls { issuer acme { ... } }` directive — see "Issue 3: ZeroSSL fallback hang" below for why.

```caddyfile
{
    email projects@foxcraft.digital
    acme_dns hetzner {env.HETZNER_API_TOKEN}
}

stella.foxcraft.digital {
    tls {
        issuer acme {
            email projects@foxcraft.digital
            dns hetzner {env.HETZNER_API_TOKEN}
        }
    }
    handle /ollama* {
        uri strip_prefix /ollama
        reverse_proxy 172.18.0.1:11434
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
            dns hetzner {env.HETZNER_API_TOKEN}
        }
    }
    reverse_proxy advoapp.finditoo.foxcraft.digital:8080
}

stella-deployment-api.foxcraft.digital {
    tls {
        issuer acme {
            email projects@foxcraft.digital
            dns hetzner {env.HETZNER_API_TOKEN}
        }
    }
    reverse_proxy deploy-api:8080
}
```

Each new subdomain gets its own block at the bottom, reverse-proxying to a container name on the `edge` network — Caddy handles TLS automatically via DNS-01, no manual cert management needed.

**Open follow-up:** `172.18.0.1:11434` (Ollama) and `172.18.0.1:8001` (stella-api) still use the Docker bridge gateway IP rather than container-name routing used by `imap-sync:3001` and `deploy-api:8080`. Functionally fine, but inconsistent with the `edge` network pattern. Low priority.

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

#### Issue 2 — Stale TXT records cause DNS-01 failures across retry attempts

Two failure patterns observed:

- `NXDOMAIN looking up TXT for _acme-challenge.<name>` — validator queried before TXT record propagated (normal on first attempt, resolves on retry).
- `Incorrect TXT record "<value>" found` — validator found a **stale** TXT value from a previous interrupted attempt. Hetzner's `rrsets` API rejects writing a second value under an existing name as `duplicate value`, so every subsequent attempt fails outright rather than overwriting.

**Root cause of the duplicates:** manually interrupting Caddy mid-attempt (`docker compose restart`, `down`, `--force-recreate`) kills the process before Caddy's own cleanup step runs, leaving a stale `_acme-challenge` TXT record for the next attempt to trip over.

**Correct procedure:**
1. Before retrying after an interrupted attempt, delete any leftover TXT record:
   ```bash
   source /opt/services/caddy/.env
   curl -s -X DELETE \
     "https://api.hetzner.cloud/v1/zones/567656/rrsets/_acme-challenge.<name>/TXT" \
     -H "Authorization: Bearer $HETZNER_API_TOKEN"
   ```
2. Start Caddy and **do not interrupt it** — let its backoff/retry cycle run unmodified. A clean first attempt typically resolves in 15–30 s once DNS has propagated; first-ever issuance for a new subdomain may take a few minutes across a few retries.
3. If still failing after ~10 minutes of uninterrupted operation, query the authoritative nameserver directly to confirm the TXT record is actually live, bypassing resolver caching:
   ```bash
   dig @ns1.your-server.de _acme-challenge.<name> TXT +short
   ```

**What was ruled out:** concurrent issuance across multiple domains (tested `stella.foxcraft.digital` in isolation — same NXDOMAIN pattern, so it's not a cross-domain race); UFW / IP whitelist (DNS-01 validation is purely outbound from Caddy to Hetzner's API, no inbound traffic involved).

#### Issue 3 — ZeroSSL fallback can hang indefinitely on a single attempt

One dns-01 attempt via ZeroSSL (Caddy's automatic fallback after a Let's Encrypt failure) produced no log output for 5+ minutes — no success, no failure, no retry-scheduled line. The DNS TXT record was correctly live the entire time, so this was not a propagation issue; the ZeroSSL ACME call itself never returned.

**Fix applied:** pinned all site blocks to Let's Encrypt only using per-site `tls { issuer acme { ... } }` (see Caddyfile above). With ZeroSSL out of the picture, all subsequent retries went through Let's Encrypt and completed normally.

**Open question:** whether this was a one-off (network hiccup, ZeroSSL-side issue) or a recurring interaction between `caddy-dns/hetzner/v2` and ZeroSSL's ACME flow isn't established — observed once. If it recurs, worth checking whether it's module-specific or a general Caddy/ZeroSSL problem.

#### Automatic Let's Encrypt staging fallback

After repeated fast failures for the same identifiers within a short window, Caddy automatically escalated to Let's Encrypt's **staging** CA (`acme-staging-v02.api.letsencrypt.org`) to avoid exhausting production rate limits. **Staging certificates are not trusted by browsers.**

In the 2026-08-07 session the staging round succeeded (proving the dns-01 pipeline worked end-to-end), then Caddy automatically retried production shortly after — confirmed by `issuer: acme-v02.api.letsencrypt.org-directory` (not `acme-staging-v02`) in the final log lines for both `advoapp.finditoo.foxcraft.digital` and `stella.foxcraft.digital`.

**Takeaway:** if you see `acme-staging-v02` in the logs, don't intervene — it's Caddy's own rate-limit protection, and it should return to production once a validation path is proven. If a staging cert ends up being served long-term (check with `curl -v` → inspect the issuer in the cert chain), something is still wrong and needs a closer look.

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
6. Add a Caddy block for the subdomain, restart Caddy
7. `docker compose build && docker compose up -d`

No UFW or `DOCKER-USER` changes needed for any of this — that's the entire point of routing everything through Caddy on the `edge` network.

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

## Ollama

- **Binary:** `/usr/local/bin/ollama`
- **Listens:** `*:11434` (all interfaces)
- **Config note:** `OLLAMA_HOST` must be set explicitly in shell (`OLLAMA_HOST=127.0.0.1:11434 ollama list`)

### Loaded Models (current as of 2026-08-06)

| Model | Size | Notes |
|-------|------|-------|
| `llama3.3:70b` | 42 GB | Primary — email drafts, high-quality generation |
| `qwen2.5:72b` | 47 GB | Under evaluation as a possible replacement for `llama3.3:70b` — better documented multilingual support (Polish not in Llama 3.3's officially supported list). Comparison on real DE/PL output still pending. |
| `qwen3:30b` | 18 GB | Kept, not in active routing |
| `gemma4:26b` | 17 GB | Kept — CPU inference is slow for this size |
| `gemma4:e4b` | 9.6 GB | Kept, not in active routing |
| `qwen2.5-coder:14b` | 9.0 GB | Kept — candidate for `code` agent (unassigned since `qwen3-coder-next` removal) |
| `qwen2.5:14b` | 9.0 GB | Classification, Slack replies |
| `gemma4:e2b` | 7.2 GB | Vision candidate for invoice/receipt extraction — not yet tested |
| `deepseek-r1:8b` | 5.2 GB | Kept, not in active routing |
| `qwen2.5-coder:7b` | 4.7 GB | Kept, not in active routing |
| `phi4-mini:latest` | 2.5 GB | Fast classification, preloaded |
| `qwen2.5:1.5b` | 986 MB | Kept, not in active routing |
| `gemma3:1b` | 815 MB | Kept, not in active routing |

**Removed 2026-08-06:** `nomic-embed-text` (only use was ChromaDB embeddings, pipeline gone), `qwen3-coder-next` (51 GB, was the `code` agent model — **removed without a replacement assigned**, open decision).

**Preloaded permanently (systemd):** `llama3.3:70b`, `phi4-mini`

---

## ChromaDB — REMOVED 2026-08-06

Fully decommissioned. See git history / prior doc versions if a vector search feature is ever rebuilt — would need ChromaDB and `nomic-embed-text` reinstalled from scratch, plus new `stella-api` routes and a new WordPress-side client, none of which exist anymore. See [`../integration/email-indexing.md`](../integration/email-indexing.md) for the full decommission notice.

---

## Security

### Current state (verified 2026-08-06; ports updated 2026-08-07)

- **UFW** — active, default-deny incoming. Ports 22, 8001, 11434, 443 restricted to `194.126.177.181` and `23.88.90.12`. Port 8001 additionally allows `172.18.0.0/16` (Docker bridge subnet) for Caddy's internal proxy calls.
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

networks:
  edge:
    external: true
```
