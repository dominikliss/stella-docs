# ddashboard and Stella server — how they fit together

This document describes the **roles**, **network paths**, **data flows**, and **operational boundaries** between the **ddashboard** WordPress theme (Hetzner managed hosting) and the **Stella** stack (Hetzner dedicated server: FastAPI, Ollama, Caddy, optional **IMAP mailbox copy** service).

> **ChromaDB removed 2026-08-06.** ChromaDB, `nomic-embed-text`, and all `/emails/*` Stella routes no longer exist. Current `stella-api` scope is chat-only. See [`email-indexing.md`](email-indexing.md) for the decommission notice.

**Related docs**

- Server topology and ports: [`../stella-server/infrastructure.md`](../stella-server/infrastructure.md)
- Stella HTTP API (FastAPI, chat-only): [`../stella-server/stella-api.md`](../stella-server/stella-api.md)
- Stella **imapsync** helper (Express via Caddy `/imap-sync`): [`../stella-server/imap-sync-service.md`](../stella-server/imap-sync-service.md)
- Email indexing pipeline (decommissioned): [`email-indexing.md`](email-indexing.md)
- WordPress theme architecture (long): [`../stella-dashboard/architecture.md`](../stella-dashboard/architecture.md)

---

## 1. Roles

| System | Role |
|--------|------|
| **ddashboard** | Custom WordPress theme: CRM, accounting, IMAP mail in MySQL (**v3** tables `dls_mail_*`), REST `dls/v1`, React SPA, AI chat agents, Ollama mail analyses. **Source of truth** for message rows, links, clients, and WP options. |
| **Stella** | Dedicated AI host: **Ollama** (LLM inference), **stella-api** (FastAPI) — **`/chat/*`** only, **Caddy** on **443**. Optional **`imap-sync`**: Express + **`imapsync`** to copy mail between two IMAP accounts (**not** ddashboard's DB import), reached only via `https://stella.foxcraft.digital/imap-sync`. ChromaDB has been uninstalled. |

Neither system replaces the other: WordPress owns relational data and sessions; Stella provides **streaming LLM inference** via `/chat/stream`.

---

## 2. Network and trust model

- **Browser** talks only to **WordPress** (HTTPS). The browser **does not** call Stella directly.
- **WordPress (ddashboard)** makes **server-to-server** HTTP to Stella for:
  - **AI chat streaming:** `POST {base}/chat/stream` (SSE, tool-calling agent)
  - **Health checks:** `GET {base}/chat/health` (`inc/routes/stella-api-test.php`)
- **Base URL** — WordPress option **`dls_stella_email_index_url`** (reused for the chat base): full HTTP root including path prefix, **no** trailing slash. Example with Caddy: `http://<stella-host>:8080/stella`.
- **Auth:** no HTTP auth on stella-api — restrict ingress at the network layer (UFW / known egress IP / VPN). Optional `dls_stella_email_index_key` may be sent as `X-Stella-Key` for future use.

---

## 3. Data flow — AI chat

```
Browser
  → POST /?dls_agent_stream=1   (SSE)
  → WordPress: load history from MySQL, build messages + tool defs
  → POST {stella_url}/chat/stream   (WordPress → Stella, curl SSE)
      loop: tool calls → MySQL services (ClientDb, MailDb, PmDb)
      until final content
  → SSE chunks streamed back to browser
```

Non-streaming / non-tool agents call Ollama directly via `POST {ollama_base_url}/api/chat` or a configured Anthropic/OpenAI provider — Stella is not involved.

---

## 4. AI features — who calls whom

| Feature | Where it runs | Backend |
|---------|----------------|---------|
| **AI chat — `general` agent (streaming)** | Browser → WordPress → Stella | **`POST /chat/stream`** on stella-api → Ollama |
| **AI chat — `general` agent (non-streaming fallback)** | WordPress → Ollama | **`POST {ollama_url}/api/chat`** directly |
| **AI chat — other agents** (accounting, clients, …) | WordPress → configured provider | Anthropic / OpenAI / Ollama per `ai_provider` setting |
| **E-Mail-AI-Analysen** (writing style, classification) | WordPress async jobs / cron | **Ollama** on configured host; corpus from mail tables |
| **Vector search / RAG over mail** | **REMOVED** | ChromaDB + `/emails/query` decommissioned 2026-08-06 |
| **Email embedding / index** | **REMOVED** | `/emails/upsert` decommissioned 2026-08-06 |

---

## 4a. Stella `imap-sync` (mailbox copy)

- **Purpose:** Operator **server-to-server IMAP copy** (`imapsync`), HTTP API for jobs/logs — not ddashboard's MySQL mail import.
- **WordPress proxy removed (2026-09-02):** `inc/routes/imap-sync-proxy.php` and the Werkzeuge E-Mail-Migration UI have been removed. The `imap-sync` service is now called directly (e.g. from Atlas or CLI).
- **Contract:** [`../stella-server/imap-sync-service.md`](../stella-server/imap-sync-service.md).

---

## 5. REST touchpoints (ddashboard)

| Area | Notes |
|------|--------|
| Mail CRUD / sync | `inc/routes/mail-*.php`, `MailSyncV2`, `MailDbService` — MySQL only, no Stella calls |
| AI chat streaming | `inc/routes/agent-chat-stream.php` → `AgentToolOrchestrator::run_streaming()` → `POST {stella}/chat/stream` |
| Stella health check | `inc/routes/stella-api-test.php` — `GET /chat/health` only; `/emails/query` probe removed |
| Options | `inc/services/option-service.php` — `dls_stella_email_index_url` (base URL), `dls_stella_email_index_key` |

Paths use WordPress REST prefix `/wp-json/dls/v1/…`.

---

## 6. Security checklist

1. **Revoke** any PAT accidentally committed; use SSH for private clones.
2. **Do not** expose Ollama or stella-api `:8001` publicly without controls; prefer Caddy + UFW.
3. **Rotate** `dls_stella_email_index_key` if ever used as a shared secret; network restriction remains primary.

---

## 7. Ops — submodule and documentation

- Canonical docs live in **`stella-docs`** (this tree), submodule from the theme: **`docs/stella-docs`**.
- When behaviour changes, update **`stella-dashboard/`**, **`stella-server/`**, and **`integration/`** together where applicable.

**Logs (Stella):** e.g. `docker logs services-stella-api-1 --tail 50` (container name may vary — see [`../stella-server/infrastructure.md`](../stella-server/infrastructure.md)).

---

## 8. Glossary

| Term | Meaning |
|------|---------|
| **ddashboard** | WordPress theme / product |
| **Stella** | Dedicated server hosting AI services |
| **stella-api** | FastAPI app — **`/chat/*`** only (email routes removed 2026-08-06) |
| **imap-sync** | Express on Stella; **`imapsync`** mailbox migration; public entry `https://stella.foxcraft.digital/imap-sync` (no host port as of 2026-09-01) |
| **gitlink** | Git submodule pointer SHA for `stella-docs` |
