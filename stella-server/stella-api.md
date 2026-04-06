# Stella API

FastAPI application running on Stella server.

- **Location:** `/opt/services/stella-api/`
- **Runtime:** Docker container (`network_mode: host`)
- **Port:** 8001 (direct) / accessible via Caddy at `:8080/stella` — ddashboard option `dls_stella_email_index_url` must be that **public base** (e.g. `http://<server>:8080/stella`), not `http://<server>:8080` alone, so paths become `…/stella/emails/document/email_123`.
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
| GET | `/emails/document/{document_id}` | ✅ Done | Chroma `POST …/collections/{id}/get` via `chroma.get_by_id`; ddashboard Kopfdaten sidebar |

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

HTTP client wrapping ChromaDB v2 API (`httpx`; upsert/query may be sync or async depending on deployment).

```python
CHROMA_URL = os.getenv("CHROMA_URL", "http://127.0.0.1:8000")
BASE = f"{CHROMA_URL}/api/v2/tenants/default_tenant/databases/default_database"
```

**Functions:**
- `get_or_create_collection(name)` → returns collection UUID `id`
- `upsert(…)` / `query(…)` → as implemented for `/emails/upsert` and `/emails/query`
- **`get_by_id(collection_id, doc_id)`** → `POST {BASE}/collections/{collection_id}/get` with `{"ids": [doc_id], "include": ["documents", "metadatas"]}` — used by **`GET /emails/document/{doc_id}`**

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

### `GET /emails/document/{document_id}`

**Contract (ddashboard `GET /dls/v1/emails/{id}/stella-chroma-raw` proxies here):**

- Path segment `document_id` is the Chroma record id, e.g. `email_24128` (same as `POST /emails/upsert` body field `id`).
- **Chroma call:** `POST {BASE}/collections/{collection_uuid}/get` with body  
  `{"ids": ["email_24128"], "include": ["documents", "metadatas"]}`  
  (same as direct Chroma v2 API on port 8000).
- Response **200** JSON (normalized single-record payload):

```json
{
  "id": "email_42",
  "document": "From: …\nSubject: …\n\n…",
  "metadata": {
    "mailbox_id": 1,
    "thread_id": "…"
  }
}
```

- Response **404** if the document is missing (`ids` empty from Chroma or falsy result).
- Embeddings are **not** returned (only `documents` + `metadatas` in the Chroma request).

#### Implementation (deployed under `/opt/services/stella-api/`)

`app/services/chroma.py` — `get_by_id`:

```python
async def get_by_id(self, collection_id: str, doc_id: str) -> dict:
    async with httpx.AsyncClient() as client:
        response = await client.post(
            f"{BASE}/collections/{collection_id}/get",
            json={"ids": [doc_id], "include": ["documents", "metadatas"]},
            timeout=30,
        )
        response.raise_for_status()
        return response.json()
```

`app/routers/emails.py`:

```python
@router.get("/document/{doc_id}")
async def get_document(doc_id: str):
    col_id = await chroma.get_or_create_collection("emails")
    result = await chroma.get_by_id(col_id, doc_id)

    if not result or not result.get("ids"):
        raise HTTPException(status_code=404, detail=f"Document {doc_id} not found")

    return {
        "id": result["ids"][0],
        "document": result["documents"][0],
        "metadata": result["metadatas"][0],
    }
```

Wire `chroma` to your app’s service instance; `get_or_create_collection` must match the **`emails`** collection used by `/emails/upsert`.

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
