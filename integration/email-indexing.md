# Email Indexing Pipeline — DECOMMISSIONED 2026-08-06

> **This entire pipeline has been removed, not just paused.** ChromaDB is uninstalled from Stella (service, data, pip package all gone). `stella-api`'s `/emails/upsert`, `/emails/query`, and `/emails/document/{id}` routes have been deleted from the codebase, along with `app/services/chroma.py` and `app/services/ollama.py`. `nomic-embed-text` has been removed from Ollama.
>
> This was already effectively dead: the ddashboard mail v3 rewrite (April 2026) had removed `StellaEmailIndexClient`, `EmailEmbedQueueService`, and `email-embed-cron.php` on the WordPress side, and the pipeline was marked "blocked" in `open-gaps.md` from that point on. No collections were ever created in ChromaDB (`/var/chromadb/data` never existed) — this feature never went live even in its pre-v3 form on the current server.
>
> **If a vector search / RAG feature over email is rebuilt in the future**, treat this document as historical reference for the intended design, but expect to rebuild the WordPress-side client, the Stella-side routes, and reinstall ChromaDB + `nomic-embed-text` from scratch — none of it exists anymore.
>
> Current `stella-api` scope: chat-only (`/chat/health`, `/chat/stream`). See [`../stella-server/stella-api.md`](../stella-server/stella-api.md).

---

## Original architecture (historical reference)

```
IMAP fetch
  → EmailCrudService::save()
    → maybe_enqueue_email_embed()     [ddashboard PHP — REMOVED in mail v3]
      → dls_email_embed_queue (MySQL) [REMOVED in mail v3]

WP-Cron (every 2 min)
  → dls_email_embed_process_cron()   [ddashboard PHP — REMOVED]
    → StellaEmailIndexClient::upsert()  [REMOVED]
      → POST /emails/upsert          [Stella API — REMOVED 2026-08-06]
        → ollama.embed()             [nomic-embed-text — REMOVED 2026-08-06]
        → chroma.upsert()            [ChromaDB — REMOVED 2026-08-06]

Reply generation:
  → POST /emails/query               [Stella API — REMOVED]
    → ollama.embed(query_text)
    → chroma.query()
    → top N emails returned
    → injected into llama3.3:70b prompt
```

The rest of this document (ddashboard-side files, request/response schemas, chunking strategy) is preserved below for historical reference only — none of it reflects the current running system.

---

## ddashboard Side (WordPress) — historical, pre-v3

### Key Files (no longer exist)

| File | Role |
|------|------|
| `inc/install-mail-tables.php` | Creates `dls_email_embed_queue` table |
| `inc/email-embed-cron.php` | WP-Cron job — `dls_email_embed_process_cron` every 2 min |
| `inc/services/email-crud-service.php` | `maybe_enqueue_email_embed()` — enqueues on IMAP save |
| `inc/services/email-embed-queue-service.php` | Queue processing, batch size setting |
| `inc/services/stella-email-index-client.php` | HTTP client — `POST /emails/upsert`, `GET /emails/document/{id}` |

### ddashboard Side — mail v3 era (April–August 2026, also removed)

`StellaEmailIndexService`, `MailStellaIndexJob`, `dls_mail_stella_index_run`, and the `POST /dls/v1/mailboxes/{id}/stella-chroma-index/*` REST routes were added as a v3 replacement client but never sent real data (Chroma was never running). As of 2026-08-06 these routes return HTTP 410 and `post_upsert()` is a no-op.

---

## Stella API Side — historical, pre-2026-08-06

### `POST /emails/upsert` (removed)
Flow was: `get_or_create_collection("emails")` → `ollama.embed(req.document)` → `chroma.upsert(...)` → `204 No Content`.

### `POST /emails/query` (removed)
Flow was: `get_or_create_collection("emails")` → `ollama.embed(req.text)` → `chroma.query(...)` → raw ChromaDB result.

### `GET /emails/document/{doc_id}` (removed)
Flow was: `chroma.get_by_id(collection_id, doc_id)` → 404 if missing, else `{id, document, metadata}`.

---

## ChromaDB Collection (historical)

- **Name:** `emails_v3` (target name at time of decommission; previously `emails` in the pre-v3 era)
- **Status:** never created — zero documents were ever indexed before the pipeline was decommissioned.
