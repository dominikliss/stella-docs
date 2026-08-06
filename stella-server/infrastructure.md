# Infrastructure

## Servers

### Stella (Hetzner Dedicated)
- **OS:** Ubuntu 24.04
- **CPU:** Intel i7-8700
- **RAM:** 124 GB
- **Role:** AI inference, API services, IMAP sync

### ddashboard (Hetzner Managed)
- **Role:** WordPress — ddashboard theme, Stella client, REST API routes

---

## Services on Stella

Docker-managed services live at `/opt/services/docker-compose.yml`.

| Service | Runtime | Port | Notes |
|---------|---------|------|-------|
| `caddy` | Docker (caddy:alpine) | 8080 (public) | Reverse proxy — routes `/ollama`, `/imap-sync`, `/stella` |
| `stella-api` | Docker (`network_mode: host`) | 8001 | FastAPI via uvicorn — **chat-only** as of 2026-08-06 |
| `imap-sync` | Docker (Node.js) | 3001 | **`imapsync` wrapper** (Express): start/status/jobs — see [`imap-sync-service.md`](imap-sync-service.md) |
| `ollama` | systemd | 11434 | LLM inference |

### Non-Docker systemd services
- `ollama.service` — `/usr/local/bin/ollama serve`
- `fail2ban.service`

**Removed 2026-08-06:** `chromadb.service` (systemd) and the `chromadb` pip package have been fully uninstalled — no ChromaDB running on Stella anymore. `github-runner` (Docker container registered against `foxcraftdigital/finditoo-contracts`) was also removed — it had never successfully registered (404 on `runner-registration`, confirmed via GitHub UI showing zero configured runners) and had no dependent workflows.

---

## Caddy Reverse Proxy

Config: `/opt/services/caddy/Caddyfile`

```
:8080 {
    /ollama*     → 172.18.0.1:11434   (Ollama)
    /imap-sync*  → imap-sync:3001     (Express → imapsync CLI — strip prefix so upstream sees /start, /status/…)
    /stella*     → 172.18.0.1:8001    (Stella API — chat routes only)
}
```

**Note:** `stella-api` uses `network_mode: host` so it reaches Ollama via `127.0.0.1` directly. Caddy routes to it via the Docker bridge gateway `172.18.0.1`.

**imap-sync:** Express listens on `/start`, `/status/:id`, `/jobs`, etc. Caddy forwards `/imap-sync` → `/` so paths match the app. Details: [`imap-sync-service.md`](imap-sync-service.md).

---

## Ollama

- **Binary:** `/usr/local/bin/ollama`
- **Listens:** `*:11434` (all interfaces)
- **Config note:** `OLLAMA_HOST` must be set explicitly in shell (`OLLAMA_HOST=127.0.0.1:11434 ollama list`) — PATH issue in some shell contexts

### Loaded Models (current as of 2026-08-06)

| Model | Size | Notes |
|-------|------|-------|
| `llama3.3:70b` | 42 GB | Primary — email drafts, high-quality generation |
| `qwen2.5:72b` | 47 GB | Under evaluation as a possible replacement for `llama3.3:70b` — better documented multilingual support (Polish not in Llama 3.3's officially supported language list; Qwen2.5 has broader multilingual training). Comparison on real DE/PL email drafts still pending. |
| `qwen3:30b` | 18 GB | Kept, not currently in active routing |
| `gemma4:26b` | 17 GB | Kept — CPU inference is slow for this size, evaluate before relying on it for latency-sensitive tasks |
| `gemma4:e4b` | 9.6 GB | Kept, not currently in active routing |
| `qwen2.5-coder:14b` | 9.0 GB | Kept — candidate for `code` agent now that `qwen3-coder-next` was removed (unassigned, see note below) |
| `qwen2.5:14b` | 9.0 GB | Classification, Slack replies, faster tasks |
| `gemma4:e2b` | 7.2 GB | Vision candidate for invoice/receipt image extraction — not yet tested against a real invoice/JPG |
| `deepseek-r1:8b` | 5.2 GB | Kept, not currently in active routing |
| `qwen2.5-coder:7b` | 4.7 GB | Kept, not currently in active routing |
| `phi4-mini:latest` | 2.5 GB | Fast classification, preloaded |
| `qwen2.5:1.5b` | 986 MB | Kept, not currently in active routing |
| `gemma3:1b` | 815 MB | Kept, not currently in active routing |

**Removed 2026-08-06:** `nomic-embed-text` (no longer needed — was only used for ChromaDB embeddings, and that pipeline is gone) and `qwen3-coder-next` (51 GB, was the documented `code` agent model — removed without a replacement assigned yet).

**Preloaded permanently (systemd):** `llama3.3:70b`, `phi4-mini`

**Two-model routing strategy:**
- `llama3.3:70b` (or `qwen2.5:72b`, pending comparison) → email drafts, high-quality generation
- `qwen2.5:14b` / `phi4-mini` → classification, faster tasks
- **`code` agent — currently unassigned.** Previously routed to `qwen3-coder-next`, which was removed 2026-08-06. Needs a decision: assign `qwen2.5-coder:14b` (already on disk) or pull a replacement.

---

## ChromaDB — REMOVED 2026-08-06

ChromaDB (previously v1.5.5, v2 API, port 8000) has been fully decommissioned:
- systemd service stopped, disabled, unit file removed
- `chromadb` pip package uninstalled (`pip3 uninstall chromadb --break-system-packages`)
- No data directory ever existed (`/var/chromadb/data` was never created — zero collections were ever indexed)
- `stella-api`'s `/emails/*` routes, `app/services/chroma.py`, and `app/services/ollama.py` (embed client) were removed from the codebase
- `CHROMA_URL` env var removed from `docker-compose.yml`

This was safe because the ddashboard-side email embed pipeline (`StellaEmailIndexClient`, `dls_email_embed_queue`, `email-embed-cron.php`) had already been removed during the mail v3 rewrite (April 2026) — see [`../integration/email-indexing.md`](../integration/email-indexing.md). Nothing was calling `/emails/upsert` or `/emails/query` anymore.

**If a vector search / RAG feature is rebuilt in the future**, ChromaDB and `nomic-embed-text` would need to be reinstalled.

---

## Security

- **UFW** — active, default-deny incoming. Ports 22, 8001, 11434 restricted to two static IPs (`194.126.177.181`, `23.88.90.12`). Port 8001 additionally allows `172.18.0.0/16` (Docker bridge subnet) so Caddy's internal reverse-proxy calls to stella-api aren't blocked.
- **fail2ban** — running
- **Caddy** — public entry point on port 8080, but see note below on Docker port publishing

### ⚠️ Docker-published ports bypass UFW

Any port published via `docker-compose.yml` (`ports: ["X:X"]`) — currently **caddy (8080)** and **imap-sync (3001)** — routes through Docker's own iptables NAT/FORWARD rules, which are evaluated **before** UFW's INPUT chain. A plain `ufw allow`/`ufw deny` rule on these ports has **no effect** regardless of what `ufw status` reports.

**Fix in place:** IP restriction for these two ports is enforced via the `DOCKER-USER` iptables chain instead, which Docker respects.

- Rules script: `/usr/local/bin/docker-user-firewall.sh`
- Applied by: `docker-user-firewall.service` (systemd oneshot, `After=docker.service`, enabled at boot)
- Rule order matters: ACCEPT rules for both IPs must be appended (`-A`, not `-I`) **before** the DROP rule for each port, or the DROP will take priority and block everyone including your own IPs.

```bash
#!/bin/bash
iptables -F DOCKER-USER

iptables -A DOCKER-USER -s 194.126.177.181 -p tcp --dport 3001 -j ACCEPT
iptables -A DOCKER-USER -s 23.88.90.12 -p tcp --dport 3001 -j ACCEPT
iptables -A DOCKER-USER -p tcp --dport 3001 -j DROP

iptables -A DOCKER-USER -s 194.126.177.181 -p tcp --dport 8080 -j ACCEPT
iptables -A DOCKER-USER -s 23.88.90.12 -p tcp --dport 8080 -j ACCEPT
iptables -A DOCKER-USER -p tcp --dport 8080 -j DROP
```

**Do not install `iptables-persistent`** to persist these rules — on this Ubuntu 24.04 setup it silently removed the `ufw` package as a conflicting dependency (both manage iptables state), which briefly left the server's normal UFW rules unenforced. The systemd-service approach above avoids this conflict entirely.

### 5678 (n8n port) — removed

A stale UFW rule allowing port 5678 (n8n's default port) from both static IPs existed with no corresponding service, container, or binary anywhere on the server. Removed 2026-08-06 — re-add only if n8n is actually deployed here.

---

## Docker Compose

Location: `/opt/services/docker-compose.yml`

```yaml
services:
  caddy:
    image: caddy:alpine
    ports:
      - "8080:8080"
    volumes:
      - ./caddy/Caddyfile:/etc/caddy/Caddyfile
    extra_hosts:
      - "host.docker.internal:host-gateway"
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
```

`CHROMA_URL` removed 2026-08-06 (no longer read by the app).
