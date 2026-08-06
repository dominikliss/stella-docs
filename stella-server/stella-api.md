# Stella API

FastAPI application on the Stella server: **chat streaming and health check only.**

> **Email routes removed 2026-08-06.** `/emails/upsert`, `/emails/query`, `/emails/document/{id}`, `/emails/message/{id}`, `/emails/collection/*`, `app/services/chroma.py`, `app/services/ollama.py`, and ChromaDB itself have all been removed from the Stella server. See [`../integration/email-indexing.md`](../integration/email-indexing.md) for the decommission notice.

- **Location:** `/opt/services/stella-api/`
- **Runtime:** Docker container (`network_mode: host`)
- **Process:** `uvicorn main:app`

**Base URL (via Caddy):** `http://<stella-host>:8080/stella`  
**Base URL (direct):** `http://<stella-host>:8001`

ddashboard option **`dls_stella_email_index_url`** holds this base URL (option name is a legacy artefact). No trailing slash in stored value.

**Auth:** none on the API — access is restricted at the network layer (UFW; allow WordPress egress IP). Optional `dls_stella_email_index_key` may be sent as `X-Stella-Key`; current Stella deploy does not require it.

**Sibling service (same host, different app):** IMAP mailbox migration via **`imapsync`** + Express on port **3001** — [`imap-sync-service.md`](imap-sync-service.md). ddashboard **Werkzeuge → E-Mail-Migration** calls it through **`inc/routes/imap-sync-proxy.php`** (not FastAPI).

---

## File structure (current)

```
stella-api/
├── Dockerfile
├── requirements.txt          # fastapi, uvicorn, httpx
└── app/
    ├── main.py               # FastAPI app, router registration
    └── routers/
        └── chat.py           # /chat routes
```

`routers/emails.py`, `services/chroma.py`, and `services/ollama.py` have been deleted.

---

## Endpoints

### `GET /chat/health`

**Response 200:** `{"status": "ok"}`

Used by ddashboard `inc/routes/stella-api-test.php` as a connectivity probe and shown in Verwaltung → AI-Anbindungen.

---

### `POST /chat/stream`

SSE streaming chat endpoint. ddashboard's `general` agent uses this as an Ollama proxy — it sends assembled messages + tool definitions and reads back ndjson chunks in real time.

**Request body**

```json
{
  "model": "llama3.3:70b",
  "messages": [ { "role": "user", "content": "…" } ],
  "tools": [ { "type": "function", "function": { "name": "…", "parameters": {} } } ],
  "think": true
}
```

**Response:** ndjson stream (one JSON object per line). Each line is one of:

```json
{ "type": "thinking", "content": "…" }
{ "type": "content",  "content": "…" }
{ "type": "tool_calls", "tool_calls": [ { "function": { "name": "…", "arguments": "…" } } ] }
{ "type": "done" }
{ "type": "error",   "detail": "…" }
```

ddashboard (`AgentToolOrchestrator::call_stella_stream()`) reads line-by-line via curl, dispatches tool calls to MySQL services, then sends results back in the next iteration until a `done` chunk is received.

---

## Environment variables

| Variable | Default | Notes |
|----------|---------|-------|
| `OLLAMA_URL` | `http://127.0.0.1:11434` | docker-compose / systemd |

`CHROMA_URL` has been removed along with the ChromaDB dependency.

---

## Known limitations / ops notes

- **No HTTP auth** — rely on firewall / VPN placement; do not expose `:8001` publicly.
- **One Ollama model in memory at a time** — streaming chat requests block while a model loads.
