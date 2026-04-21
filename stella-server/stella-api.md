# Stella API

FastAPI application on the Stella server: chat healthcheck, **mail v3** email indexing into ChromaDB (`emails_v3`), and semantic query.

- **Location:** `/opt/services/stella-api/`
- **Runtime:** Docker container (`network_mode: host`)
- **Process:** `uvicorn main:app`

**Base URL (via Caddy):** `http://<stella-host>:8080/stella`  
**Base URL (direct):** `http://<stella-host>:8001`

ddashboard option **`dls_stella_email_index_url`** must be the **HTTP root** used before `/emails/…` / `/chat/…` — e.g. `http://<host>:8080/stella` so WordPress calls `…/stella/emails/upsert`. Trailing slashes are stripped client-side.

**Auth:** none on the API — access is restricted at the network layer (UFW; operator docs: allow WordPress egress / ProtonVPN static IP as applicable). An optional WordPress option `dls_stella_email_index_key` may still be sent as `X-Stella-Key` for future use; the current Stella deploy does not require it.

**Sibling service (same host, different app):** IMAP mailbox migration via **`imapsync`** + Express on port **3001** — [`imap-sync-service.md`](imap-sync-service.md). ddashboard **Werkzeuge → E-Mail-Migration** calls it through **`inc/routes/imap-sync-proxy.php`** (not FastAPI).

---

## File structure (typical)

```
stella-api/
├── Dockerfile
├── requirements.txt          # fastapi, uvicorn, httpx
└── app/
    ├── main.py               # FastAPI app, router registration
    ├── routers/
    │   ├── chat.py           # /chat routes
    │   └── emails.py         # /emails routes
    └── services/
        ├── chroma.py         # ChromaDB client (v2 API)
        └── ollama.py         # Ollama embed client (nomic-embed-text)
```

---

## ChromaDB collection

| Name | Role |
|------|------|
| **`emails_v3`** | One Chroma document per **`dls_mail_message_link`** row (or a **sentinel** document when there are no links). Same embedding vector is reused for all link-documents of one message. |

---

## Endpoints

### Chat

| Method | Path | Response |
|--------|------|----------|
| GET | `/chat/health` | **200** `{"status": "ok"}` |

---

### Emails — `POST /emails/upsert`

Indexes one row from **`dls_mail_message`**. Before insert, **deletes** all existing Chroma documents for that `message_id` (so removed links are cleaned up automatically). Creates **one document per entry** in `links`. If `links` is empty, creates a **sentinel** document with `entity_type="none"` (document id `msg_{message_id}_link_0`).

The embedding is computed **once** per request and applied to **all** link-documents for that message.

**Request body**

```json
{
  "message_id": 123,
  "document": "Subject: Betreff\n\nBody text (quote-stripped)",

  "account_id": 1,
  "folder_id": 4,
  "direction": "inbound",
  "date_sent_ts": 1712345678,
  "date_received_ts": 1712345700,
  "thread_id": "hdr-abc123",
  "category": "invoice_question",
  "action": "reply_needed",
  "urgency": "immediate",
  "classification_source": "rule",
  "has_attachment": 0,
  "is_seen": 1,
  "is_flagged": 0,

  "links": [
    {
      "link_id": 55,
      "entity_type": "client",
      "entity_id": 42,
      "link_source": "manual"
    },
    {
      "link_id": 56,
      "entity_type": "project",
      "entity_id": 7,
      "link_source": "folder"
    }
  ]
}
```

**Enumerated / constrained values**

| Field | Values |
|------|--------|
| `direction` | `"inbound"` / `"outbound"` |
| `classification_source` | `"rule"` / `"ai"` / `""` |
| `has_attachment`, `is_seen`, `is_flagged` | `0` / `1` |
| `entity_type` (in `links`) | `"client"` / `"project"` / `"none"` |
| `link_source` | `"folder"` / `"manual"` / `"address"` / `"none"` |
| `category`, `action`, `urgency` | English tokens from `dls_mail_message`; `""` if not classified |

**Message without links**

```json
{
  "message_id": 124,
  "document": "Subject: …\n\n…",
  "links": []
}
```

→ Creates document id **`msg_124_link_0`** with `entity_type="none"`, `entity_id=0`.

**Response:** **204 No Content**

---

### Emails — `DELETE /emails/message/{message_id}`

Deletes **all** Chroma documents for that message. Call when a row is removed from **`dls_mail_message`**.

**Response:** **204 No Content**

---

### Emails — `POST /emails/query`

Semantic similarity over indexed messages.

**Request body**

```json
{
  "text": "Suchtext",
  "where": { "$and": [{"entity_type": {"$eq": "client"}}, {"entity_id": {"$eq": 42}}] },
  "n_results": 10
}
```

- `where` — optional. Chroma metadata filter syntax (`$eq`, `$and`, `$or`, …). Pass `null` or omit for an unfiltered search.
- `n_results` — optional, default **10**. Max is enforced by Chroma (collection size).

**⚠️ One result per Chroma document, not per message.**  
Because each message produces one document per link, the same `message_id` can appear multiple times in `results` (once per link-document that matches). Callers that want one result per message must deduplicate on `metadata.message_id`, keeping the hit with the lowest `distance`.

**Response 200**

```json
{
  "results": [
    {
      "id": "msg_123_link_55",
      "document": "Subject: Betreff\n\nBody text…",
      "metadata": {
        "message_id": 123,
        "account_id": 1,
        "folder_id": 4,
        "direction": "inbound",
        "date_sent_ts": 1712345678,
        "date_received_ts": 1712345700,
        "thread_id": "hdr-abc123",
        "category": "invoice_question",
        "action": "reply_needed",
        "urgency": "immediate",
        "classification_source": "rule",
        "has_attachment": 0,
        "is_seen": 1,
        "is_flagged": 0,
        "link_id": 55,
        "entity_type": "client",
        "entity_id": 42,
        "link_source": "manual"
      },
      "distance": 0.21
    }
  ]
}
```

`distance` — ChromaDB cosine distance; **smaller = more similar**. Typical good hits are below **0.3**. Range is [0, 2].

> **Implementation note:** ChromaDB returns a list-of-lists format internally; `_normalize_query_result` in `routers/emails.py` flattens this into the `results` array above before returning.

**`where` examples**

Filter by client (use `entity_type` + `entity_id` together — there is no `client_id` metadata field):

```json
{ "$and": [{"entity_type": {"$eq": "client"}}, {"entity_id": {"$eq": 42}}] }
```

Filter by direction:

```json
{ "direction": {"$eq": "inbound"} }
```

Filter by thread:

```json
{ "thread_id": {"$eq": "hdr-abc123"} }
```

Filter by mailbox account:

```json
{ "account_id": {"$eq": 1} }
```

Filter by specific message (useful to check what is indexed):

```json
{ "message_id": {"$eq": 123} }
```

Combined — client inbound only:

```json
{
  "$and": [
    {"entity_type": {"$eq": "client"}},
    {"entity_id": {"$eq": 42}},
    {"direction": {"$eq": "inbound"}}
  ]
}
```

---

### Emails — `GET /emails/document/{doc_id}`

Single Chroma document by full id.

**Path format:** `msg_{message_id}_link_{link_id}`  
**Example:** `GET /emails/document/msg_123_link_55`

**Response 200:**

```json
{
  "id": "msg_123_link_55",
  "document": "Subject: Betreff\n\nBody text…",
  "metadata": { "message_id": 123, "entity_type": "client", "entity_id": 42, "…": "…" }
}
```

**Response 404:** document not found (Chroma returned no IDs for that doc id).

---

### Emails — `GET /emails/message/{message_id}`

All Chroma documents for one message (one per link, or sentinel).

**Example:** `GET /emails/message/123`

**Response 200:**

```json
{
  "message_id": 123,
  "documents": [
    { "id": "msg_123_link_55", "document": "…", "metadata": { } },
    { "id": "msg_123_link_56", "document": "…", "metadata": { } }
  ]
}
```

**Response 404:** no documents for that `message_id`.

---

### Emails — `GET /emails/collection/count`

**Response 200:** `{ "collection": "emails_v3", "count": 412 }`

---

### Emails — `POST /emails/collection/reset`

Drops and recreates the **`emails_v3`** collection — **all** indexed data is lost.

**Response:** **204 No Content**

---

## Chroma document IDs

| Situation | ID |
|-----------|-----|
| Message with links | `msg_{message_id}_link_{link_id}` |
| Message without links (sentinel) | `msg_{message_id}_link_0` |

Legacy ids like `email_{id}` are **not** used in `emails_v3`.

---

## Metadata fields (every document)

All fields are present and filterable via `where`.

| Field | Type | Source |
|------|------|--------|
| `message_id` | int | `dls_mail_message.id` |
| `account_id` | int | `dls_mail_message.account_id` |
| `folder_id` | int | `dls_mail_message.folder_id` |
| `direction` | string | `dls_mail_message.direction` |
| `date_sent_ts` | int | `dls_mail_message.date_sent` (Unix) |
| `date_received_ts` | int | `dls_mail_message.date_received` (Unix) |
| `thread_id` | string | `dls_mail_message.thread_id` |
| `category` | string | `dls_mail_message.email_category` |
| `action` | string | `dls_mail_message.email_action` |
| `urgency` | string | `dls_mail_message.email_urgency` |
| `classification_source` | string | `dls_mail_message.classification_source` |
| `has_attachment` | int | `dls_mail_message.has_attachment` |
| `is_seen` | int | `dls_mail_message.is_seen` |
| `is_flagged` | int | `dls_mail_message.is_flagged` |
| `link_id` | int | `dls_mail_message_link.id` (**0** = no link / sentinel) |
| `entity_type` | string | `dls_mail_message_link.entity_type` |
| `entity_id` | int | `dls_mail_message_link.entity_id` |
| `link_source` | string | `dls_mail_message_link.source` |

---

## Services (implementation)

### `services/ollama.py`

Embeddings via Ollama **`/api/embeddings`**, model **`nomic-embed-text`** (typical env: `OLLAMA_URL=http://127.0.0.1:11434`).

### `services/chroma.py`

ChromaDB **v2** HTTP API (`CHROMA_URL`, e.g. `http://127.0.0.1:8000`), tenant/database path as in [`infrastructure.md`](infrastructure.md).

---

## Environment variables

| Variable | Default | Notes |
|----------|---------|-------|
| `OLLAMA_URL` | `http://127.0.0.1:11434` | docker-compose / systemd |
| `CHROMA_URL` | `http://127.0.0.1:8000` | docker-compose |

---

## Known limitations / ops notes

- **No chunking** — one embedding per message text, replicated across link-documents; very long bodies depend on `nomic-embed-text` context limits (may truncate).
- **No HTTP auth** — rely on firewall / VPN placement; do not expose `:8001` or Chroma/Ollama publicly without controls.
- **Reset** — `POST /emails/collection/reset` is destructive; use only for rebuilds.

---

## ddashboard PHP (integration sketch)

**Option:** `dls_stella_email_index_url` — e.g. `http://<host>:8080/stella`

**Upsert** (`StellaEmailIndexService::post_upsert()`)

```php
$base = rtrim( (string) get_option( 'dls_stella_email_index_url', '' ), '/' );
$url  = $base . '/emails/upsert';
$response = wp_remote_post( $url, [
	'headers' => [ 'Content-Type' => 'application/json' ],
	'body'    => wp_json_encode( $payload ),
	'timeout' => 30,
] );
// HTTP 204 = success
```

**Delete (on `dls_mail_message` delete)**

```php
$url = $base . '/emails/message/' . (int) $message_id;
$response = wp_remote_request( $url, [ 'method' => 'DELETE', 'timeout' => 15 ] );
```

**Semantic search (`POST /dls/v1/emails/semantic-search` in ddashboard)**

ddashboard proxies the query and deduplicates results by `message_id` (since Stella returns one hit per Chroma document, not per message). It also converts `distance` [0, 2] to a `semantic_score` [0, 1]:

```php
// score = 1 - distance/2   (0.0 = no match, 1.0 = perfect)
$score = max( 0.0, min( 1.0, 1.0 - $distance / 2.0 ) );
```

Only the best-scoring document per `message_id` is kept. The resolved `dls_mail_message` rows are returned with a `semantic_score` field appended.

WordPress must build `$payload` from **`dls_mail_message`** + **`dls_mail_message_link`** rows (see request schema above). Queue/cron wiring is product-side — see [`../integration/email-indexing.md`](../integration/email-indexing.md).
