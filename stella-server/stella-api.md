# Stella API

FastAPI application running on Stella server.

- **Location:** `/opt/services/stella-api/`
- **Runtime:** Docker container (`network_mode: host`)
- **Port:** 8001 (direct) / accessible via Caddy at `https://stella.foxcraft.digital/stella`
- **Process:** `uvicorn app.main:app`
- **Scope as of 2026-08-06: chat-only.** The `/emails/*` routes and ChromaDB integration were removed — see below.

**Sibling service (same host, different app):** IMAP mailbox migration via **`imapsync`** + Express on port **3001** — [`imap-sync-service.md`](imap-sync-service.md). ddashboard **Werkzeuge → E-Mail-Migration** calls it through **`inc/routes/imap-sync-proxy.php`** (not FastAPI).

ddashboard option **`dls_stella_email_index_url`** holds the base URL (option name is a legacy artefact from the email indexing era). No trailing slash in stored value. **⚠️ Follow-up outstanding (2026-08-07):** this option still points at the old `http://<ip>:8080/stella` URL and must be updated to `https://stella.foxcraft.digital/stella`.

**Auth:** none on the API — access is restricted at the network layer (UFW / known egress IP). Optional `dls_stella_email_index_key` may be sent as `X-Stella-Key`; current Stella deploy does not require it.

---

## File Structure (current)

```
stella-api/
├── Dockerfile
├── requirements.txt          # fastapi, uvicorn, httpx, pydantic — no chromadb dependency
└── app/
    ├── main.py               # FastAPI app, router registration (chat only)
    └── routers/
        ├── __init__.py
        └── chat.py           # /chat/health, /chat/stream
```

**Removed 2026-08-06:**
- `app/routers/emails.py` — implemented `/emails/upsert`, `/emails/query`, `/emails/document/{id}`
- `app/services/chroma.py` — ChromaDB v2 API HTTP client
- `app/services/ollama.py` — Ollama `/api/embeddings` client (only consumer was `emails.py`)

Reason: ddashboard's mail v3 rewrite (April 2026) had already removed `StellaEmailIndexClient` and the entire embed queue pipeline on the WordPress side — nothing was calling these routes anymore. ChromaDB was fully uninstalled from Stella the same day (see [`infrastructure.md`](infrastructure.md)).

---

## Endpoints

### Chat

| Method | Path | Status | Notes |
|--------|------|--------|-------|
| GET | `/chat/health` | ✅ Active | Returns `{"status": "ok"}` |
| POST | `/chat/stream` | ✅ Active | SSE proxy to Ollama `/api/chat`. Supports `think` (thinking-token streaming) and `tools` (tool-call passthrough). Used by ddashboard's `general` agent as the primary streaming path. |

### Emails — REMOVED

All `/emails/*` routes were removed 2026-08-06. See [`../integration/email-indexing.md`](../integration/email-indexing.md) for the full decommission notice and historical schema.

If a vector search / RAG feature is rebuilt in the future, these routes (and ChromaDB and `nomic-embed-text`) would need to be reinstated from git history.

---

## `POST /chat/stream` — request/response

```json
// Request
{
  "model": "phi4-mini",
  "messages": [{"role": "user", "content": "..."}],
  "think": false,
  "tools": null
}
```

Response: newline-delimited JSON chunks (`text/event-stream`):

```
{"type": "thinking", "content": "..."}     — thinking token (only when think=true)
{"type": "content",  "content": "..."}     — response token
{"type": "tool_call", "tool_calls": [...]} — tool call request
{"type": "done"}                            — stream finished
{"type": "error", "detail": "..."}          — upstream error
```

---

## Known call paths (ddashboard `general` agent)

Per ddashboard's `AgentToolOrchestrator`, the `general` agent has two paths:

1. **Streaming** (`run_streaming`) — WordPress → `POST {stella_url}/chat/stream` (curl SSE) → Stella proxies to Ollama.
2. **Non-streaming fallback** (`run`) — WordPress → Ollama directly via `POST {ollama_base_url}/api/chat` (`wp_remote_post`), bypassing Stella entirely.

Both paths verified working end-to-end 2026-08-06, including through Caddy and against the corrected UFW rules for ports 8001/8080/11434.
