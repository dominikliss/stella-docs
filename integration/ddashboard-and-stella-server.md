# ddashboard and Stella server — AI chat path

This document is the **ddashboard ↔ Stella AI** contract (chat stream, Ollama, what WordPress owns). It is **not** the company-wide map.

**For Atlas + ddashboard + Stella + Edison together**, start at [`system-overview.md`](system-overview.md).

This page describes the **roles**, **network paths**, **data flows**, and **operational boundaries** between the **ddashboard** WordPress theme (Hetzner managed hosting) and the **Stella** stack (Hetzner dedicated server: FastAPI, Ollama, Caddy). IMAP mailbox **copy** is operated from Atlas, not from this theme.

> **ChromaDB removed 2026-08-06.** ChromaDB, `nomic-embed-text`, and all `/emails/*` Stella routes no longer exist. Current `stella-api` scope is chat-only. See [`email-indexing.md`](email-indexing.md) for the decommission notice.

**Related docs**

- **Full stack (includes Atlas):** [`system-overview.md`](system-overview.md)
- Server topology and ports: [`../stella-server/infrastructure.md`](../stella-server/infrastructure.md)
- Stella HTTP API (FastAPI, chat-only): [`../stella-server/stella-api.md`](../stella-server/stella-api.md)
- Stella **imapsync** helper (now driven by Atlas): [`../stella-server/imap-sync-service.md`](../stella-server/imap-sync-service.md), [`../atlas/tools.md`](../atlas/tools.md)
- Email indexing pipeline (decommissioned): [`email-indexing.md`](email-indexing.md)
- WordPress theme architecture (long): [`../stella-dashboard/architecture.md`](../stella-dashboard/architecture.md)
- Atlas identity (ddashboard tokens): [`../atlas/auth.md`](../atlas/auth.md)

---

## 1. Roles

| System | Role |
|--------|------|
| **ddashboard** | Custom WordPress theme: CRM, accounting, IMAP mail in MySQL (**v3** tables `dls_mail_*`), REST `dls/v1`, React SPA, AI chat agents, Ollama mail analyses. **Source of truth** for message rows, links, clients, and WP options. Also the **identity provider** for Atlas (`/dls/v1/auth/*`). |
| **Stella** | Dedicated AI host: **Ollama** (LLM inference), **stella-api** (FastAPI) — **`/chat/*`** only, **Caddy** on **443**. **`imap-sync`**, **deploy-api**, and **health-api** live here too but are called by **Atlas**, not by this theme. ChromaDB has been uninstalled. |
| **Atlas** | Ops platform (separate Laravel app). Not on this AI path. See [`system-overview.md`](system-overview.md). |

ddashboard and Stella do not replace each other: WordPress owns relational data and sessions; Stella provides **streaming LLM inference** via `/chat/stream`. Atlas is a third app that reuses ddashboard login and Stella’s ops APIs.

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
- **WordPress proxy removed (2026-09-02):** `inc/routes/imap-sync-proxy.php` and the Werkzeuge E-Mail-Migration UI have been removed. Atlas **Tools → IMAP Migration** is the UI (`../atlas/tools.md`).
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
- When behaviour changes, update **`stella-dashboard/`**, **`stella-server/`**, **`atlas/`**, and **`integration/`** together where applicable.

**Logs (Stella):** e.g. `docker logs services-stella-api-1 --tail 50` (container name may vary — see [`../stella-server/infrastructure.md`](../stella-server/infrastructure.md)).

---

## 8. Glossary

| Term | Meaning |
|------|---------|
| **ddashboard** | WordPress theme / product; Atlas identity provider |
| **Atlas** | Laravel ops platform — [`../atlas/README.md`](../atlas/README.md) |
| **Stella** | Dedicated server hosting AI and ops services |
| **stella-api** | FastAPI app — **`/chat/*`** only (email routes removed 2026-08-06) |
| **imap-sync** | Express on Stella; **`imapsync`** mailbox migration; called from Atlas; public entry `https://stella.foxcraft.digital/imap-sync` |
| **gitlink** | Git submodule pointer SHA for `stella-docs` |
