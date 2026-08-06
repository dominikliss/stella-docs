# Open Gaps & Next Steps

Last updated: 2026-08-06

**Cross-system overview:** [integration/ddashboard-and-stella-server.md](integration/ddashboard-and-stella-server.md)

---

## Email Indexing Pipeline — DECOMMISSIONED

> ChromaDB, `nomic-embed-text`, and all `/emails/*` Stella routes were removed on 2026-08-06. This gap is closed by removal, not implementation. See [`integration/email-indexing.md`](integration/email-indexing.md) for the full decommission notice and historical architecture.

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

- PM task indexing into a vector store (if rebuilt in the future)
- Codebase/Bitbucket indexing
- Action Scheduler (using WP-Cron for now)
- n8n workflows for email/Slack message-to-task extraction
- Fine-tuning `qwen2.5:14b` on writing style (ruled out — using prompt-based style profiles instead)
- KSeF-compliant automated accounting entries via Stella
- Slack assistant for client Q&A
