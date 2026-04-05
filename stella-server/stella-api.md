# Stella API

FastAPI application running on Stella server.

- **Location:** `/opt/services/stella-api/`
- **Runtime:** Docker container (`network_mode: host`)
- **Port:** 8001 (direct) / accessible via Caddy at `:8080/stella`
- **Process:** `uvicorn main:app`

---

## File Structure

```
stella-api/
├── Dockerfile
├── requirements.txt          # fastapi, uvicorn, httpx
└── app/
    ├── main.py               # FastAPI app, router registration
    ├── routers/
    │   ├── __init__.py
    │   ├── chat.py           # /chat routes
    │   └── emails.py         # /emails routes
    └── services/
        ├── __init__.py
        ├── chroma.py         # ChromaDB client (v2 API)
        └── ollama.py         # Ollama embed client
```

---

## Endpoints

### Chat

| Method | Path | Status | Notes |
|--------|------|--------|-------|
| GET | `/chat/health` | ✅ Done | Returns `{"status": "ok"}` |

### Emails

| Method | Path | Status | Notes |
|--------|------|--------|-------|
| POST | `/emails/upsert` | ✅ Done | Embeds document + upserts into ChromaDB |
| POST | `/emails/query` | ✅ Done | Similarity search with optional `where` filter |

---

## Services

### `services/ollama.py`

Async HTTP client wrapping Ollama `/api/embeddings`.

```python
OLLAMA_URL = os.getenv("OLLAMA_URL", "http://127.0.0.1:11434")

async def embed(text: str) -> list[float]:
    # POST /api/embeddings
    # model: nomic-embed-text
    # returns: embedding vector
```

**Model used:** `nomic-embed-text`
**Timeout:** 60s

### `services/chroma.py`

Sync HTTP client wrapping ChromaDB v2 API.

```python
CHROMA_URL = os.getenv("CHROMA_URL", "http://127.0.0.1:8000")
BASE = f"{CHROMA_URL}/api/v2/tenants/default_tenant/databases/default_database"
```

**Functions:**
- `get_or_create_collection(name)` → returns collection `id`
- `upsert(collection_id, id, embedding, document, metadata)` → upserts single doc
- `query(collection_id, embedding, where, n_results)` → returns raw ChromaDB response

---

## Request / Response Schemas

### `POST /emails/upsert`

```json
// Request
{
  "id": "email_123",
  "document": "From: foo@bar.com\nSubject: ...\n\nbody text...",
  "metadata": {
    "sender": "foo@bar.com",
    "direction": "inbound",
    "date": 1712345678,
    "subject": "..."
  }
}

// Response: 204 No Content
```

### `POST /emails/query`

```json
// Request
{
  "text": "query string",
  "where": { "sender": "foo@bar.com" },
  "n_results": 5
}

// Response
{
  "results": { ...raw ChromaDB query response... }
}
```

---

## Environment Variables

| Variable | Default | Notes |
|----------|---------|-------|
| `OLLAMA_URL` | `http://127.0.0.1:11434` | Set in docker-compose |
| `CHROMA_URL` | `http://127.0.0.1:8000` | Set in docker-compose |

---

## Known Issues / Open Items

- `POST /emails/upsert` returns `204` but does **not** return the upserted ID — ddashboard currently gets no confirmation beyond HTTP status
- `chroma.query()` returns raw ChromaDB response structure — not normalized for ddashboard consumption yet
- No chunking logic — long emails are sent as single document; if body exceeds `nomic-embed-text` context window (~8192 tokens) it will be silently truncated by Ollama
- No auth on any endpoint — relies on UFW/network-level security
- `where: {}` passed as empty dict from ddashboard when no filter — needs to be handled as `None` in chroma query (currently done in router: `req.where or None`)
