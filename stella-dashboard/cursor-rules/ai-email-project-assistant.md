# Cursor rule export: `ai-email-project-assistant.mdc`

_Source of agent rule in theme repo: `.cursor/rules/ai-email-project-assistant.mdc`._

---

# AI Email & Project Assistant — ddashboard Implementation Overview

> **Roadmap / vision** — Ollama, ChromaDB, `email_accounts`, embed pipeline, Slack. Most of this is **not** implemented in the theme yet.

## Implemented today in the theme (IMAP “Nachrichten”)

The theme already ships **MySQL tables** `dls_mailbox`, `dls_email`, spam blocklist/whitelist, IMAP sync (`MailSyncService`, `MailChunkSyncService`), REST `dls/v1` routes in `inc/routes/mailboxes.php`, and UI at `/nachrichten`. **Documentation:** `docs/mail-nachrichten.md`, `.cursor/rules/mail-nachrichten.mdc`, and **Nachrichten (IMAP)** in `architecture.mdc`. That stack is separate from the **AI/Chroma** pipeline described below; migrate or align when building the full spec.

**Verwaltung → AI-Analyse:** Zusätzlich gibt es **E-Mail-AI-Analysen** auf Basis gespeicherter `dls_email`-Zeilen: **Schreibstil** (Ollama, ausgehend) und **Klassifizierungs-Taxonomie** (Ollama, eingehend, Markdown inkl. regelbasierte Stufe). Siehe **`architecture.mdc`** (Abschnitt *E-Mail-AI-Analysen*) und optional **`.cursor/rules/email-ai-analysis.mdc`**.

## What this system is (target)

A self-hosted AI assistant integrated into ddashboard (custom-built, MySQL-backed) that:

- Reads inbound emails and drafts replies in the owner's voice for review
- Categorises and tags incoming emails automatically
- Extracts structured data (contacts, invoice references) into the CRM
- Answers client questions via Slack automatically, scoped to their projects only
- Gives the owner an AI assistant with access to full history: emails, tasks, code, and accounting

All inference runs on a **separate dedicated model server** running **Ollama**. Models: **`llama3.3:70b`** (email drafts), **`qwen2.5:14b`** (Slack + classification), **`qwen3-coder-next`** (technical/code queries), **`nomic-embed-text`** (embeddings). The ddashboard server handles orchestration, auth, and data access. **ChromaDB** runs on the model server alongside Ollama.

---

## Infrastructure overview

```
ddashboard server (MySQL + PHP)          Model server
─────────────────────────────────        ──────────────────────────────────
- ddashboard app                         - Ollama (llama3.3:70b)      ← email drafts
- Email inbound handler                  - Ollama (qwen2.5:14b)       ← Slack + classification
- Sync worker (cron)                     - Ollama (qwen3-coder-next)  ← technical / code queries
- Context builder                        - Ollama (nomic-embed-text)  ← all embeddings
- Review queue UI                        - ChromaDB 1.5.5 (v2 API, port 8000)
- Slack bot receiver
- Role-based access control
```

Communication between ddashboard and model server: **internal HTTP only**. Neither Ollama nor ChromaDB is exposed publicly.

---

## Data sources and storage

### Source of truth: MySQL (ddashboard server)

All data lives here permanently. ChromaDB is a search index on top of it, never the primary store.

| Table / domain | Notes |
|----------------|-------|
| `emails` | Raw email content, headers, metadata |
| `tasks` | PM tasks, status, assignee, time logs |
| `task_comments` | Comments on tasks |
| `invoices` | Accounting — owner-only access |
| `contacts` | CRM contacts linked to clients |
| `client_project_access` | Maps clients to their allowed project IDs |
| `client_slack_identities` | Maps Slack user IDs to client records |
| `embed_queue` | Pending sync rows for ChromaDB |
| `chroma_sync_log` | Tracks which rows have been embedded + their chroma_id |

### Vector index: ChromaDB (model server)

Three collections; all include `project_id` and `client_id` in metadata for filtering:

| Collection | Content | Chunk strategy | Sync trigger |
|------------|---------|----------------|--------------|
| `emails` | Email body + subject | One chunk per email; ~500 tokens for long emails | Every 5–10 min via embed_queue |
| `tasks` | Task description + all comments concatenated | One chunk per task | Nightly batch + on-save |
| `codebase` | PHP/JS/TS source | One chunk per function/method; whole file for configs | On git push via webhook |

**Accounting data (invoices, payroll, salaries) never goes into ChromaDB.** Always queried directly from MySQL with role-based access checks.

---

## Multi-account email + contact resolver

### `email_accounts` table

All IMAP accounts stored in ddashboard DB. Poller iterates active accounts every **5 minutes**.

```sql
CREATE TABLE email_accounts (
    id           BIGINT AUTO_INCREMENT PRIMARY KEY,
    address      VARCHAR(255) NOT NULL,
    display_name VARCHAR(255),
    imap_host    VARCHAR(255) NOT NULL,
    imap_port    INT DEFAULT 993,
    username     VARCHAR(255) NOT NULL,
    password     VARCHAR(255) NOT NULL,  -- encrypted at rest
    is_active    TINYINT DEFAULT 1,
    last_polled_at TIMESTAMP NULL
);
```

### Contact resolver

Every inbound email passes through contact resolution before processing:

1. Exact match on `contact_email_addresses.email`
2. Domain match on `contact_email_addresses.domain`
3. No match → `needs_triage = 1`, queued for manual assignment in triage UI

### `emails` table — extra columns

```sql
ALTER TABLE emails ADD COLUMN account_id       BIGINT NOT NULL;
ALTER TABLE emails ADD COLUMN contact_id       BIGINT NULL;
ALTER TABLE emails ADD COLUMN project_id       BIGINT NULL;
ALTER TABLE emails ADD COLUMN folder_original  VARCHAR(255);
ALTER TABLE emails ADD COLUMN needs_triage     TINYINT DEFAULT 0;
ALTER TABLE emails ADD COLUMN spam_score       FLOAT NULL;
ALTER TABLE emails ADD COLUMN is_spam          TINYINT DEFAULT 0;
ALTER TABLE emails ADD COLUMN has_attachment   TINYINT DEFAULT 0;
```

### Spam + injection protection

- Spam emails (score > 5.0) stored in MySQL but **never** added to `embed_queue`
- HTML stripped before embedding — **plain text only** into ChromaDB
- Email content wrapped in `<email>` tags in prompt — model told to treat as **untrusted data**
- Attachment content **never** embedded — only email body text

---

## embed_queue pattern

Every INSERT or UPDATE on `emails`, `tasks`, `task_comments` writes a row to `embed_queue`. The sync worker drains this table on a schedule, generates embeddings via Ollama, pushes to ChromaDB, and writes the `chroma_id` back to the source row.

```sql
CREATE TABLE embed_queue (
  id          BIGINT AUTO_INCREMENT PRIMARY KEY,
  source_type ENUM('email', 'task', 'comment') NOT NULL,
  source_id   BIGINT NOT NULL,
  operation   ENUM('upsert', 'delete') DEFAULT 'upsert',
  created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  INDEX (created_at)
);
```

---

## Email pipeline (inbound)

### Trigger

**IMAP polling** (managed email hoster — no Dovecot/Sieve access). A cron job on ddashboard polls all configured email accounts via IMAP every **5 minutes**, fetches **UNSEEN** messages, and dispatches async jobs for processing. Each account stored in **`email_accounts`** with IMAP credentials.

### Processing steps

1. **Inbound handler** — parse raw email (headers, body; threading via `Message-ID` / `In-Reply-To`), store in `emails`, write to `embed_queue`
2. **Context builder** — SQL + Chroma (thread, contact CRM, similar emails, tasks, code if technical, accounting only for `owner`)
3. **Ollama call** — POST `/api/chat` on model server
4. **Result processor** — structured JSON:

```json
{
  "draft_reply": "...",
  "category": "invoice_question",
  "tags": ["client", "invoice", "urgent"],
  "extracted_data": { "invoice_number": "INV-2024-042" },
  "confidence": 0.87
}
```

5. **Output actions** — draft to review queue, tags on email, CRM extraction

### Review queue

Owner approves / edits / discards; SMTP send on approve; optional auto-send by category + confidence threshold.

---

## Codebase sync (Bitbucket)

1. Webhook → `git pull` on mirror → `git diff` → chunk by logical unit → upsert/delete Chroma `codebase` collection
2. **Auth:** scoped token (App Passwords deprecated June 2026)
3. **Ignore:** `vendor/`, `node_modules/`, `*.lock`, `*.env`, compiled assets

**Markdown + cursor rules in repo:**

- Markdown chunked by heading section (`##`, `###`), not line count
- **`.cursorrules` / `.mdc`** tagged `type: cursor_rule` in metadata — always included for technical queries regardless of similarity
- Metadata `type`: `function`, `class`, `file`, `documentation`, `cursor_rule`

---

## Slack client assistant

- Events API webhook; bot only in designated per-client channels
- Flow: resolve sender → `client_slack_identities` → `client_project_access` → **all Chroma queries use `WHERE project_id IN (allowed_ids)`** at query layer → Ollama → Slack post
- System prompt: scoped client + project names only; no pricing/finances; refuse out-of-scope
- Isolation: Chroma `where` + MySQL `project_id IN (?)` + prompt + internal-only Ollama/Chroma

---

## Role-based access

| Role | Email drafts | Tasks | Codebase | Accounting | Slack assistant |
|------|-------------|-------|----------|------------|-----------------|
| `owner` | full | ✓ | ✓ | ✓ SQL only | n/a |
| `employee` | limited | ✓ | ✓ | never | n/a |
| `client` | ✗ | read-only scoped | read-only scoped | ✗ | ✓ scoped |

Access checks in **context builder** before any fetch or model call.

---

## Writing style

Static **style profile** (from sent-email analysis) in system prompt; editable field in ddashboard; prepended to email draft prompts. No client data baked into weights.

---

## Model routing

| Task | Model | Reason |
|------|--------|--------|
| Email draft generation | `llama3.3:70b` | Highest quality; async OK |
| Slack replies | `qwen2.5:14b` | Faster, conversational |
| Email classification | `qwen2.5:14b` | Small task, speed |
| Technical / code | `qwen3-coder-next` | Code-optimised |
| All RAG embeddings | `nomic-embed-text` | Fast, 768-dim |

Same Ollama instance; **one model in memory at a time** — Slack jobs can get higher queue priority vs 70b email drafts.

---

## Implementation phases (checklist)

### Phase 1 — Foundation

- [ ] `embed_queue` + hooks on relevant saves
- [ ] Sync worker (cron 5–10 min): queue → Ollama embed → Chroma → `chroma_id`
- [ ] ChromaDB client (PHP HTTP, **v2 API** `/api/v2/`)
- [ ] Ollama client (`/api/chat`, `/api/embed`)
- [ ] Historical email + task import (one-time)
- [ ] **`email_accounts`** + management UI
- [ ] **`contact_email_addresses`** (multiple addresses per contact)
- [ ] Contact resolver (exact → domain → triage)
- [ ] Email triage UI (unknown senders → contacts / no-reply)

### Phase 2 — Email pipeline

- [ ] **IMAP poller cron** — every account in `email_accounts`, UNSEEN → `ProcessInboundEmail` job
- [ ] Inbound parse/store + `embed_queue`
- [ ] Threading headers
- [ ] Classifier
- [ ] Context builder + Ollama JSON + review queue + auto-send + CRM extractor

### Phase 3 — Codebase sync

- [ ] Bitbucket webhook, git pull, chunker, embed/upsert, delete vectors

### Phase 4 — Slack assistant

- [ ] Slack app, webhook, `client_slack_identities` + `client_project_access` UIs, scoped builder, reply sender, fallback

### Phase 5 — Owner accounting assistant

- [ ] Accounting SQL only; owner gate; owner UI

### Configuration / admin

- [ ] Style profile editor
- [ ] Sync status dashboard
- [ ] Review queue / auto-send settings
- [ ] Client access manager (Slack + projects)

---

## Key constraints

- Ollama + Chroma on **model server** — ddashboard calls internal HTTP only
- ChromaDB **no row-level ACL** — isolation in ddashboard before query
- **Accounting never in ChromaDB**
- Bitbucket: scoped tokens (not App Passwords after June 2026)
- Threading: **Message-ID headers**, not subject
- Code chunks: logical units (function/class), not line count
- Slack: reject unknown users **before** model call
- No fine-tuning — style is prompt-only
- **ChromaDB 1.5.5 — v2 API** — use `/api/v2/`, not `/api/v1/`
- **IMAP polling** (managed hoster) — no Sieve/Dovecot; **multiple accounts** via `email_accounts`

---

## Relation to current theme code

The theme implements **`dls_mailbox` / `dls_email`** (plus spam lists), **webklex/php-imap** — see **`docs/mail-nachrichten.md`**. The roadmap above targets **`email_accounts`**, embed queue, ChromaDB, and Ollama — **merge or replace** when the AI pipeline is wired in.
