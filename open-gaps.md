# Open Gaps & Next Steps

Last updated: 2026-04-21

**Pipeline detail:** [integration/email-indexing.md](integration/email-indexing.md)  
**Cross-system overview:** [integration/ddashboard-and-stella-server.md](integration/ddashboard-and-stella-server.md)

---

## Email Indexing Pipeline

> **Stella (done in docs / target deploy):** FastAPI indexes **`dls_mail_message`** into Chroma collection **`emails_v3`** — `POST /emails/upsert` (multi-doc per link + sentinel), `DELETE /emails/message/{id}`, normalized **`POST /emails/query`**, collection count/reset, document/message GET. See [`stella-server/stella-api.md`](stella-server/stella-api.md) and [`integration/email-indexing.md`](integration/email-indexing.md).

> **ddashboard (still open):** Legacy `dls_email` queue and `StellaEmailIndexClient` were removed with mail v3. WordPress must enqueue, build payloads from **`dls_mail_message`** + **`dls_mail_message_link`**, call upsert/delete, and optionally expose debug REST against `GET /emails/document/msg_*`.

### WordPress / theme

- [ ] **Design embed queue for v3** — column on `dls_mail_message`, separate queue table, or Action Scheduler; keep `integration/email-indexing.md` in sync
- [x] **HTTP client + batch upsert** — `StellaEmailIndexService` + `POST /dls/v1/mailboxes/{id}/stella-chroma-index` (Verwaltung → Nachrichten, per-account Chroma action)
- [ ] **Retry cap** — ~3 attempts + `last_error` (or AS failure log) on remote failures
- [ ] **Hook DELETE** — when a `dls_mail_message` row is removed, call Stella `DELETE /emails/message/{message_id}`
- [ ] **Update `stella-api-test.php` + AI-Anbindungen UI** — health/query probes match **`emails_v3`** contract (document ids `msg_*_link_*`)

### Medium priority (Stella or both)

- [ ] **Global indexing pause flag** — stop enqueuing during Stella deploys / model swaps
- [ ] **Length guard** — optional character cap in Stella upsert with log warning
- [ ] **Chunking** — only if single-vector truncation becomes a problem; would extend id/metadata scheme beyond one vector per message

---

## Infrastructure

- [ ] **GitHub migration** — move all repos from Bitbucket to GitHub
  - ddashboard
  - stella-api (currently not in version control — needs a repo)
  - finditoo (5 consumer repos + contracts)
- [ ] **stella-api in git** — `/opt/services/stella-api` is not version controlled yet
- [ ] **Daily connectivity tests** — automated login/fetch/update tests for contract sync with failure notifications (stack not finalized)

---

## Deferred (not in current scope)

- PM task indexing into ChromaDB
- Codebase/Bitbucket indexing
- Generic `stella_embed_queue` schema
- Action Scheduler (using WP-Cron for now)
- n8n workflows for email/Slack message-to-task extraction
- Fine-tuning `qwen2.5:14b` on writing style (ruled out — using prompt-based style profiles instead)
- KSeF-compliant automated accounting entries via Stella
- Slack assistant for client Q&A
