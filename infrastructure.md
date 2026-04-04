# Infrastructure

## Servers

### Stella (Hetzner Dedicated)
- **OS:** Ubuntu 24.04
- **CPU:** Intel i7-8700
- **RAM:** 124 GB
- **Role:** AI inference, vector DB, API services, IMAP sync

### ddashboard (Hetzner Managed)
- **Role:** WordPress — ddashboard theme, Stella client, REST API routes

---

## Services on Stella

All services run via Docker Compose at `/opt/services/docker-compose.yml`.

| Service | Runtime | Port | Notes |
|---------|---------|------|-------|
| `caddy` | Docker (caddy:alpine) | 8080 (public) | Reverse proxy — routes /ollama, /imap-sync, /stella |
| `stella-api` | Docker (network_mode: host) | 8001 | FastAPI via uvicorn |
| `imap-sync` | Docker (Node.js) | 3001 (internal) | IMAP sync service |
| `chromadb` | systemd | 8000 | Vector database (v1.5.5, v2 API) |
| `ollama` | systemd | 11434 | LLM inference |

### Non-Docker systemd services
- `ollama.service` — `/usr/local/bin/ollama serve`
- `chromadb.service`
- `fail2ban.service`
- `github-runner` (Docker container) — registered against `foxcraftdigital/finditoo-contracts`

---

## Caddy Reverse Proxy

Config: `/opt/services/caddy/Caddyfile`

```
:8080 {
    /ollama*     → 172.18.0.1:11434   (Ollama)
    /imap-sync*  → imap-sync:3001     (IMAP sync service)
    /stella*     → 172.18.0.1:8001    (Stella API)
}
```

**Note:** `stella-api` uses `network_mode: host` so it reaches Ollama and ChromaDB via `127.0.0.1` directly. Caddy routes to it via the Docker bridge gateway `172.18.0.1`.

---

## Ollama

- **Binary:** `/usr/local/bin/ollama`
- **Listens:** `*:11434` (all interfaces)
- **Config note:** `OLLAMA_HOST` must be set explicitly in shell (`OLLAMA_HOST=127.0.0.1:11434 ollama list`) — PATH issue in some shell contexts

### Loaded Models

| Model | Size | Notes |
|-------|------|-------|
| `llama3.3:70b` | 42 GB | Primary — email drafts, high-quality generation |
| `qwen2.5:72b` | 47 GB | Large general model |
| `qwen3:30b` | 18 GB | — |
| `qwen3-coder-next:latest` | 51 GB | MoE — ~3B active params, fast on CPU |
| `qwen2.5-coder:14b` | 9.0 GB | Code tasks |
| `qwen2.5:14b` | 9.0 GB | Classification, faster tasks |
| `qwen2.5-coder:7b` | 4.7 GB | — |
| `deepseek-r1:8b` | 5.2 GB | — |
| `phi4-mini:latest` | 2.5 GB | Fast classification, preloaded |
| `nomic-embed-text:latest` | 274 MB | Embeddings — used for ChromaDB indexing |
| `qwen2.5:1.5b` | 986 MB | — |
| `gemma3:1b` | 815 MB | — |

**Preloaded permanently (systemd):** `llama3.3:70b`, `phi4-mini`

**Two-model routing strategy:**
- `llama3.3:70b` → email drafts, high-quality generation
- `qwen2.5:14b` / `phi4-mini` → classification, faster tasks

---

## ChromaDB

- **Version:** 1.5.5
- **API:** v2 (v1 returns deprecation errors — always use `/api/v2/...`)
- **Port:** 8000
- **Base path:** `/api/v2/tenants/default_tenant/databases/default_database`
- **Collections:** none yet (as of 2026-04-05)

---

## Security

- **UFW** — only ProtonVPN static IP whitelisted for Ollama/ChromaDB direct access
- **fail2ban** — running
- **Caddy** — public entry point on port 8080

---

## Docker Compose

Location: `/opt/services/docker-compose.yml`

```yaml
services:
  caddy:
    image: caddy:alpine
    ports: ["8080:8080"]
    volumes: [./caddy/Caddyfile:/etc/caddy/Caddyfile]
    extra_hosts: ["host.docker.internal:host-gateway"]
    restart: unless-stopped

  imap-sync:
    build: ./imap-sync
    restart: unless-stopped

  stella-api:
    build: ./stella-api
    network_mode: host
    environment:
      - OLLAMA_URL=http://127.0.0.1:11434
      - CHROMA_URL=http://127.0.0.1:8000
    restart: unless-stopped
```
