# Open Gaps & Next Steps

Last updated: 2026-04-19

**Pipeline detail:** [integration/email-indexing.md](integration/email-indexing.md)  
**Cross-system overview:** [integration/ddashboard-and-stella-server.md](integration/ddashboard-and-stella-server.md)

---

## Email Indexing Pipeline

> **Status:** The ddashboard mail stack was rewritten to v3 (`DLS_MAIL_DB_VERSION = 31`). The legacy `dls_email` table, `EmailEmbedQueueService`, `StellaEmailIndexClient`, and `email-embed-cron.php` no longer exist. The embed pipeline described in `integration/email-indexing.md` needs to be rebuilt against the new `dls_mail_message` / `dls_mail_message_link` schema before any of the items below apply.

### Blocked until v3 integration is implemented

- [ ] **Design embed queue for v3** — decide whether to add an `embed_queue` column on `dls_mail_message`, a separate queue table, or use Action Scheduler; update `integration/email-indexing.md` accordingly
- [ ] **Re-implement indexing client** — new `StellaEmailIndexClient` (or equivalent) pointing at `dls_mail_message` rows
- [ ] **Retry cap** — cap at ~3 attempts; write `last_error` somewhere on failure
- [ ] **Normalize query response** — `POST /emails/query` currently returns raw ChromaDB structure; normalize for easier consumption in ddashboard

### Medium Priority (after v3 integration)

- [ ] **Global indexing pause flag** — settings toggle to stop enqueuing without touching code (useful during Stella deploys / model swaps)
- [ ] **Length guard** — add character cap (~6000 chars) in Stella upsert with truncation warning in logs
- [ ] **Chunking** — implement `email_{id}_0`, `email_{id}_1` pattern in Stella if long emails become a problem

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
