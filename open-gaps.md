# Open Gaps & Next Steps

Last updated: 2026-04-05

**Pipeline detail:** [integration/email-indexing.md](integration/email-indexing.md)  
**Cross-system overview:** [integration/ddashboard-and-stella-server.md](integration/ddashboard-and-stella-server.md)

---

## Email Indexing Pipeline

### High Priority

- [ ] **End-to-end test** — trigger manual re-index for one email, verify document lands in ChromaDB `emails` collection
- [ ] **Verify retry cap** — confirm `email-embed-queue-service.php` caps at ~3 attempts and writes `last_error` on failure
- [ ] **Verify metadata fields** — confirm `build_metadata()` in `StellaEmailIndexClient` includes: `sender`, `direction`, `date` (unix int), `subject`, `email_id`

### Medium Priority

- [ ] **Normalize query response** — `POST /emails/query` currently returns raw ChromaDB structure; normalize for easier consumption in ddashboard
- [ ] **Global indexing pause flag** — settings toggle to stop enqueuing without touching code (useful during Stella deploys / model swaps)

### Low Priority

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
