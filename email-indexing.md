# Email Indexing Pipeline

End-to-end flow for indexing emails from ddashboard into ChromaDB via Stella API.

**Status: partially complete — see gaps below.**

---

## Architecture

```
IMAP fetch
  → EmailCrudService::save()
    → maybe_enqueue_email_embed()     [ddashboard PHP]
      → dls_email_embed_queue (MySQL)

WP-Cron (every 2 min)
  → dls_email_embed_process_cron()   [ddashboard PHP]
    → StellaEmailIndexClient::upsert()
      → POST /emails/upsert          [Stella API]
        → ollama.embed()             [nomic-embed-text]
        → chroma.upsert()            [ChromaDB emails collection]

Reply generation:
  → POST /emails/query               [Stella API]
    → ollama.embed(query_text)
    → chroma.query()                 [similarity search + where filter]
    → top N emails returned
    → injected into llama3.3:70b prompt
```

---

## ddashboard Side (WordPress)

### Key Files

| File | Role |
|------|------|
| `inc/install-mail-tables.php` | Creates `dls_email_embed_queue` table |
| `inc/email-embed-cron.php` | WP-Cron job — `dls_email_embed_process_cron` every 2 min |
| `inc/services/email-crud-service.php` | `maybe_enqueue_email_embed()` — enqueues on IMAP save |
| `inc/services/email-embed-queue-service.php` | Queue processing, batch size setting |
| `inc/services/stella-email-index-client.php` | HTTP client — `POST /emails/upsert`, `build_metadata()` |
| `inc/routes/mail-email-embed.php` | REST: `POST /dls/v1/emails/{id}/embed-index` (manual re-index) |
| `inc/routes/stella-api-test.php` | REST: health probe + query test endpoints |

### Queue Table: `dls_email_embed_queue`

```sql
email_id     INT UNIQUE   -- FK to dls_email
attempts     INT          -- retry counter
last_error   TEXT         -- last failure message
```

### Document Format

One document per email sent to Stella:

```
From: sender@example.com
Subject: Subject line

Body text (quote-stripped)
```

### Metadata Fields (from `build_metadata()`)

Verify these match what Stella's `where` filters expect:

| Field | Type | Notes |
|-------|------|-------|
| `sender` | string | Email address |
| `direction` | string | `inbound` / `outbound` |
| `date` | int | Unix timestamp |
| `subject` | string | Email subject |
| `email_id` | int | Raw MySQL ID for cross-referencing |

### Enqueue Logic

`maybe_enqueue_email_embed()` enqueues when:
- Embed is enabled in settings
- Email is not spam

### Retry Logic

- Max attempts cap: **verify in `email-embed-queue-service.php`** (should be ~3)
- Failed calls must increment `attempts` and write `last_error`
- Emails hitting the cap should stop retrying silently

---

## Stella API Side

### `POST /emails/upsert`

✅ Implemented in `app/routers/emails.py`

Flow:
1. `get_or_create_collection("emails")`
2. `ollama.embed(req.document)` — model: `nomic-embed-text`
3. `chroma.upsert(col_id, req.id, embedding, req.document, req.metadata)`
4. Returns `204 No Content`

**Stable ID contract:** ddashboard sends `id: "email_{id}"` — Stella uses this as ChromaDB document ID.

### `POST /emails/query`

✅ Implemented in `app/routers/emails.py`

Flow:
1. `get_or_create_collection("emails")`
2. `ollama.embed(req.text)`
3. `chroma.query(col_id, embedding, where or None, n_results)`
4. Returns raw ChromaDB result

---

## ChromaDB Collection

- **Name:** `emails`
- **Created:** lazily on first upsert via `get_or_create_collection()`
- **Status:** not yet created (no emails indexed as of 2026-04-05)

---

## Chunking Strategy

Currently: **one document per email, no chunking.**

- `nomic-embed-text` context window: ~8192 tokens
- Long emails will be silently truncated by Ollama if they exceed this
- If chunking is needed later: implement on Stella side using `email_{id}_0`, `email_{id}_1`, etc.
- ddashboard does not need to change — chunking is Stella's responsibility

---

## Open Gaps

| Gap | Where | Priority |
|-----|-------|----------|
| Verify retry cap (3 attempts) + `last_error` write in queue service | ddashboard | High |
| Verify `build_metadata()` fields match Stella `where` filter fields | ddashboard | High |
| Settings flag to globally pause indexing (for Stella deploys) | ddashboard | Medium |
| Normalize `POST /emails/query` response for ddashboard consumption | Stella | Medium |
| Add length guard for very long emails (e.g. 6000 char cap + truncation warning) | Stella | Low |
| End-to-end test: send one email through full pipeline, verify in ChromaDB | Both | High |

---

## End-to-End Test (first run)

Once queue and Stella are connected:

```bash
# 1. Trigger manual re-index for one email from ddashboard REST
POST /wp-json/dls/v1/emails/{id}/embed-index

# 2. Check Stella logs
docker logs services-stella-api-1 --tail 50

# 3. Verify collection created + document in ChromaDB
curl http://localhost:8000/api/v2/tenants/default_tenant/databases/default_database/collections
curl http://localhost:8000/api/v2/tenants/default_tenant/databases/default_database/collections/{col_id}/count

# 4. Test query
curl -X POST http://localhost:8001/emails/query \
  -H "Content-Type: application/json" \
  -d '{"text": "test query", "n_results": 3}'
```
