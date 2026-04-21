# Email indexing pipeline

End-to-end story for indexing **mail v3** messages from ddashboard (MySQL **`dls_mail_message`** / **`dls_mail_message_link`**) into ChromaDB via **Stella API**, collection **`emails_v3`**.

**Authoritative HTTP reference:** [`../stella-server/stella-api.md`](../stella-server/stella-api.md) (paths, payloads, metadata, document ids).

---

## Status (April 2026)

| Layer | State |
|--------|--------|
| **Stella API** | Contract documented: `POST /emails/upsert`, `DELETE /emails/message/{id}`, `POST /emails/query`, document/message/collection helpers, collection **`emails_v3`**. |
| **ddashboard** | **`StellaEmailIndexService`** + async job **`POST /dls/v1/mailboxes/{id}/stella-chroma-index/start`** → **`…/step`** (browser loop, like chunk IMAP import). Run audit: **`dls_mail_stella_index_run`**. Verwaltung → Nachrichten: per-account **Chroma-Index** (sparkles). **DELETE** on message removal still optional. Options **`dls_stella_email_index_url`** / **`dls_stella_email_index_key`** (AI-Anbindungen). |

---

## Architecture (target)

```
IMAP / sync
  → dls_mail_message (+ dls_mail_message_link)
      → [future] enqueue or Action Scheduler job
          → HTTP POST {base}/emails/upsert     [Stella API]
            → ollama.embed(document)         [nomic-embed-text, once per message]
            → chroma upsert emails_v3        [one doc per link; sentinel if no links]

On dls_mail_message DELETE:
  → HTTP DELETE {base}/emails/message/{message_id}

Semantic search / RAG:
  → HTTP POST {base}/emails/query
      → normalized { results: [ { id, document, metadata, distance } ] }
```

**Base URL** `dls_stella_email_index_url`: same string you would put before `/emails/upsert` in curl, e.g. **`http://<stella-host>:8080/stella`** (Caddy) or **`http://<stella-host>:8001`** (direct uvicorn on port 8001). No path injection in client code — concatenate `{base}/emails/…`.

---

## Stella behaviour (summary)

- **Upsert** replaces all Chroma rows for that `message_id` first, then writes **one document per link** with ids `msg_{message_id}_link_{link_id}`. Empty `links` → single sentinel **`msg_{message_id}_link_0`** with `entity_type="none"`, `link_id=0`.
- **One embedding** per upsert, shared across all link-documents for that message.
- **Document text** — convention: `Subject: …\n\n` + quote-stripped body (built in WordPress before POST).
- **Query** returns a **normalized** `results` array (not raw Chroma wire format); each hit includes **`distance`** (cosine distance; lower = closer).

---

## ddashboard responsibilities (to implement)

1. **Build payload** from SQL:
   - Top-level fields from **`dls_mail_message`** (ids, timestamps as Unix, `thread_id`, classification columns, flags).
   - **`links`** array from **`dls_mail_message_link`** (`link_id`, `entity_type`, `entity_id`, `link_source` mapped from column `source`).
2. **Upsert** after sync or classification updates when indexing is enabled.
3. **DELETE** Stella-side documents whenever **`dls_mail_message`** is deleted in MySQL (hard delete or purge), so Chroma does not retain orphans.
4. **Optional:** persist “last indexed” on the message row (schema TBD — column on `dls_mail_message` vs separate queue table vs Action Scheduler).

---

## PHP integration (sketch)

Matches [`stella-server/stella-api.md`](../stella-server/stella-api.md):

```php
$base = rtrim( (string) get_option( 'dls_stella_email_index_url', '' ), '/' );
$response = wp_remote_post( $base . '/emails/upsert', [
	'headers' => [ 'Content-Type' => 'application/json' ],
	'body'    => wp_json_encode( $payload ),
	'timeout' => 30,
] );
// 204 = success

wp_remote_request( $base . '/emails/message/' . (int) $message_id, [
	'method'  => 'DELETE',
	'timeout' => 15,
] );
```

---

## ChromaDB

- **Collection name:** **`emails_v3`**
- **Direct Chroma debug** (operator): v2 API on port **8000** — see [`../stella-server/infrastructure.md`](../stella-server/infrastructure.md).

---

## Chunking

Still **one logical embedding per message** (vector copied to each link-document). If bodies exceed the embed model context, handle truncation or future chunking **on Stella**; link-document ids would remain per link for the first chunk or gain a chunk suffix if a multi-chunk design is added later.

---

## Open items (product)

| Gap | Owner |
|-----|--------|
| Queue / scheduler design for v3 (column vs table vs Action Scheduler) | ddashboard |
| `StellaEmailIndexClient` (or equivalent) + hooks from mail sync / classification | ddashboard |
| REST: manual re-index + “raw Chroma” debug using **`GET …/emails/document/msg_*`** or **`GET …/emails/message/{id}`** | ddashboard |
| Align **AI-Anbindungen** copy and health checks with **`emails_v3`** paths and response shapes | ddashboard (`inc/routes/stella-api-test.php`) |

---

## End-to-end test (operator)

```bash
BASE=http://<host>:8080/stella   # or :8001

curl -sS "$BASE/chat/health"

curl -sS -X POST "$BASE/emails/upsert" \
  -H "Content-Type: application/json" \
  -d '{"message_id":999,"document":"Subject: Test\n\nBody","account_id":1,"folder_id":1,"direction":"inbound","date_sent_ts":1,"date_received_ts":2,"thread_id":"t","category":"","action":"","urgency":"","classification_source":"","has_attachment":0,"is_seen":1,"is_flagged":0,"links":[]}'

curl -sS "$BASE/emails/message/999"

curl -sS -X POST "$BASE/emails/query" \
  -H "Content-Type: application/json" \
  -d '{"text":"Test","n_results":3}'
```
