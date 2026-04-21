# ddashboard — Capabilities Overview

ddashboard is a self-built internal business application for **Dominik Liss**, a self-employed web developer and entrepreneur based in Vienna, Austria. It runs as a custom WordPress theme with a React frontend compiled via webpack, backed by the WordPress REST API and a custom `dls/v1` REST namespace.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Backend | PHP 8, WordPress (custom theme) |
| Frontend | React, Formik + Yup, `@wordpress/core-data` |
| Build | webpack → `build/index.js` |
| Styles | SCSS (ScssPhp, compiled on theme load) |
| Custom Fields | ACF Pro (JSON config in `inc/acf/settings/`) |
| Database | MySQL (WordPress tables + 6 custom table groups) |
| PDF generation | Spipu Html2Pdf / TCPDF |
| Email / IMAP | webklex/php-imap (via Composer) |
| AI | Anthropic Claude API (claude-sonnet-4-6) |
| Exchange rates | NBP (Polish National Bank) public API |
| Time tracking | TrackingTime Pro API v4 |
| E-invoicing | KSeF (Polish Ministry of Finance, API v2 — test + production) |
| Accounting export | SaldeoSMART API |
| YouTube | YouTube Data API v3 + YouTube Analytics API |

---

## Modules

### 1. Accounting (Buchhaltung) — `/` dashboard root

The core financial module. Handles income and expenses for a sole proprietorship operating in Austria and Poland.

**Transactions (income & expenses)**
- Record income (`Einnahme`) and expenses (`Ausgabe`) with net amount, currency, dates, VAT, and category
- Currencies: EUR, PLN, USD (USD only for expenses)
- Primary date: `invoice_date` (accounting date); secondary: `transaction_date` (payment date)
- VAT modes: standard (`nein`), Reverse Charge, Innengemeinschaftlicher Erwerb, Drittland
- Expense categories: Werbung, Material, Beratung, GWG, Investition, Erlöskorrektur
- Automatic PLN/EUR conversion rates fetched from the NBP API and stored per transaction
- KSeF reference and seller NIP stored as ACF fields for Polish e-invoice traceability (expense list shows a **KSeF** badge when `ksef_reference` is set; income list shows the same badge when the linked invoice has KSeF send status **sent** or **pending**, or a non-empty `ksef_invoice_number`)

**Invoices**
- Invoices linked to a client and a transaction
- Multi-language: DE, EN, PL
- Bank account chosen from a configurable table (`dls_bank_account`); primary account for the invoice currency is auto-selected based on the linked transaction; can be overridden per invoice
- Line items (positions) with product references, quantity, unit price, and VAT
- PDF generation from the invoice form; versioned PDFs stored on disk

**Bank Accounts**
- Custom `dls_bank_account` table for managing multiple bank accounts (AT/EUR, PL/PLN, USA/USD and more)
- Each account has: label, IBAN/BIC or US routing numbers, account holder, condition filters (country, VAT rate), currency, and an `is_primary` flag per currency
- Intelligent three-tier selection: invoice pin → transaction pin → `BankAccountDbService::find_account()` (currency first, then country/VAT tiebreaker)
- Managed via Verwaltung → Buchhaltung → bank account panel (side-by-side with KSeF settings)

**Products (Produkte)**
- Product catalogue with base price, description, recurring flag, and subscription period (monthly / quarterly / yearly)
- Optional client link (client-specific products)
- Subscription Billing runs automatically via WP-Cron (daily): generates invoices and income transactions for all active subscriptions

**Accounting Year Config (Jahresabschluss)**
- Per-year configuration: vacation / sick-day budget, emergency fund, investment budget, advance tax payments, open costs
- Quarterly VAT payment tracking (four checkboxes per year)

**Accounting stats & chart**
- Date-range filter (month/year picker)
- Income vs. expense chart with category breakdown
- Open invoices panel (invoices without a linked transaction)

**KSeF integration (Polish e-invoices)**
- Import Polish purchase invoices from the KSeF API (Ministry of Finance)
- Send outgoing invoices to KSeF as FA(3) XML (schema `1-0E`) via encrypted interactive sessions (mandatory from 2026-02-01 for large enterprises, 2026-04-01 for all)
- Sending flow: JWT auth → open session with AES-256 key (RSA-OAEP encrypted) → encrypt invoice (AES-256-CBC) → submit → close session → poll status
- Supports domestic PL buyers (23% VAT), EU reverse-charge (np I), and third-country exports (np II)
- **Separate tokens** for test (`ksef_test_token`) and production (`ksef_token`) environments; `KSeFService` selects automatically based on `ksef_test_mode` option
- Test environment: `https://ap-test.ksef.mf.gov.pl/` — generate test token at the KSeF test portal
- Production environment: `https://ap.ksef.mf.gov.pl/`
- Re-sending: invoices can be re-sent to a different environment (tracked via `_ksef_sent_env` post meta); re-sending to the same environment when already sent is blocked (HTTP 409)
- 6-step async JWT authentication (RSA-OAEP-SHA256), same token for import and send sessions
- Fixed 3-month import window (idempotent upsert)
- Upsert logic: create writes all fields; re-import only updates financial/date fields — never overwrites user-edited notes
- Send tracking: KSeF number, send status (pending/sent/error), error messages, session+invoice references, sent environment stored on invoice CPT
- XML validation before send (structural checks against FA(3) schema requirements)
- FA(3) mandatory markers always included: `JST`/`GV` in Podmiot2; `NoweSrodkiTransportu`/`P_23`/`PMarzy` in Adnotacje
- Send button always visible in invoice list — backend controls allow/block logic
- Automatic PLN conversion rate via NBP
- Seller name rule: invoices from "Good Code" → note set to "Freelancer (Pawel)"

**Saldeo export**
- One-click CSV/XML export for the tax advisor in SaldeoSMART format
- Configured under Verwaltung → Saldeo (API key stored in WP options)

**NBP exchange rates**
- `GET /dls/v1/nbp-rate` — EUR/PLN mid rate for a given date
- `GET /dls/v1/nbp-expense-rates` — EUR/PLN + USD/PLN + derived USD→EUR rate for expense recording
- Results cached with WordPress transients (7 days)

---

### 2. Client Management (Kunden)

Full CRM for managing clients, contacts, documents, and portal access.

**Client records**
- Status: `active`, `lead`, `inactive`
- Stammdaten: official name, website, email, phone, address, ZIP, city, country, VAT number, registration number
- Multiple invoice email addresses (repeater)
- Logo (media library)
- Social links (repeater)

**Contact persons (Personen)**
- Each client can have multiple linked persons (separate CPT)
- Fields: name, phone, email, birthday, profile image
- Person card with inline edit/delete

**Documents (Dateien / file CPT)**
- Types: `document`, `proposal`, `contract`
- Document subtypes: `normal`, `terms_of_service` (AGB)
- Contract subtypes: `normal`, `wp_maintenance`
- Body editor: TipTap (rich text, StarterKit)
- Template system: `is_template` flag; files can derive content from a named template (`content_mode: from_template`)
- Template-driven title: auto-set from template name + client website domain
- PDF generation: versioned PDFs (`-v1`, `-v2`, …) stored on disk; `public_url` updated after generation
- TipTap HTML and legacy Gutenberg markup both supported for PDF rendering
- PDF acceptance tracking: `accepted_date`, `accepted_method`, `accepted_ip_address`

**Client portal**
- Creates a WordPress user with `customer` role for the client
- Username derived from client name; strong random password generated once and returned
- Portal URL: `https://dominikliss.com/kundenbereich/`
- AGB auto-creation: if a file template titled exactly "Allgemeine Geschäftsbedingungen" exists, a client-specific AGB document is created and a PDF generated automatically on portal user creation
- `DELETE /dls/v1/client-portal-user` removes only the linked customer WP user; client data untouched
- Blocks all WordPress notification emails for `customer` role users (new-user, password-change, lost-password)

**Referral / commission system**
- Mark clients as referred (`referred_client`) with a referrer client link
- Commission percent (default 10%), payout date cutoff
- PDF commission reports generated and stored persistently (SHA-256 filename, no overwrite)
- Versioned report list per referrer client (ACF repeater)
- `GET /dls/v1/referral-commission-report/{id}` — generates and returns the PDF
- `DELETE /dls/v1/referral-commission-report/{id}/{row}` — removes a specific report row and its PDF file

---

### 3. Project Management (Projekte) — `/projects`

A built-in PM board, optionally synced with TrackingTime Pro.

**Structure**
- Projects → Task lists → Tasks → Comments + Time entries
- Projects linked to a client (`client_id`)
- Tasks have title, status, due date, sort order, and assignees (multiple WP users)

**TrackingTime Pro integration**
- Import projects, tasks, task lists, and time entries from TrackingTime Pro API v4
- Idempotent import via `tt_project_id` / `tt_task_id` stored on each record
- Configurable via Verwaltung → Projekte (API key in WP options)

**Custom MySQL tables** (option `dls_pm_db_version`):
`dls_pm_project`, `dls_pm_task_list`, `dls_pm_task`, `dls_pm_task_assignee`, `dls_pm_comment`, `dls_pm_time_entry`

**REST routes** under `dls/v1/pm/`:
Projects, task lists, tasks, assignees, comments, time entries — full CRUD.

---

### 4. Nachrichten (IMAP Email Inbox) — `/nachrichten`

A custom email inbox that syncs multiple IMAP accounts into MySQL **v3** tables (`dls_mail_db_version`, `DLS_MAIL_DB_VERSION = 32`). The `/nachrichten` screen uses **`PageLayout`** with a single **main column** (max-width), a dark **inbox card**, **`.client-data` + `.tabs`** for **Kunden / WordPress / Sonstiges** (same DOM pattern as the client card), thread rows via **`MessagesFocusInboxRow`**, and **`TablePagination`**. List data loads with **`useDlsQuery`** (`GET /dls/v1/emails` with `imap_folder=INBOX` and pagination — the REST layer still supports additional filters, but the inbox UI does not expose a filter/search bar). Loading states use **inbox skeleton** components (`SkeletonInboxTabsStrip`, `SkeletonInboxFocusRows`, `SkeletonInboxPaginationBar`). Opening a thread opens a wide **`SidebarForm`** (`messages-conversation-sidebar.js`): sender, **client badge**, **subject below sender** (aligned with the list), **`ScrollPanel`** + **`EmailConversationThread`**, mailto reply draft; no separate “Schnellzugriff” column.

**IMAP sync**
- WP-Cron every 5 minutes (`dls_mail_imap_sync_cron`, `dls_every_five_minutes`) — **`MailSyncV2::sync_account`** per active account (default batch size 100, filterable via `dls_mail_cron_sync_limit`, minimum 10)
- **Chunk** wipe-resync for large rebuilds (progress in Verwaltung → Nachrichten)
- webklex/php-imap (no PHP `ext-imap` required)
- IMAP passwords encrypted at rest (`MailCrypto`)

**Email storage** (v3 — see `stella-dashboard/mail-nachrichten.md` for the full table list)
- Core: `dls_mail_account`, `dls_mail_folder`, `dls_mail_message` (includes `thread_id`, direction, classification fields when enabled, `is_seen`, `has_attachment`, …)
- Links: `dls_mail_folder_link`, `dls_mail_message_link` (`source`: folder / address / manual)
- Attachments: `dls_mail_attachment`; files under `wp-content/private/dls-mail-attachments/{account_id}/{message_id}/`
- Classification rules + sync audit: `dls_mail_classification_rule`, `dls_mail_sync_run`

**Client assignment** (message links, not a single `client_id` column on the legacy model)
1. **Folder** — folder→entity from `dls_mail_folder_link` (highest priority)
2. **Address** — From/To matching against CRM email maps (`MailDbService`)
3. **Manual** — user-set via `PUT /dls/v1/emails/{id}`; preserved across wipe-resync via transient restore

**Optional AI classification** — `MailClassificationService`, gated by `DLS_MAIL_CLASSIFICATION_ENABLED` (default off); Layer-1 rules + optional Layer-2 Ollama for main INBOX inbound only.

**Thread / conversation view**
- Threads grouped by `thread_id` on `dls_mail_message`; chronology and bubbles in **`EmailConversationThread`** / **`EmailMessageBlock`**

**Attachments**
- Stored privately; download URLs exposed via the mail emails REST layer (see `mail-emails.php`)

**Admin (Verwaltung → Nachrichten)**
- Mailbox CRUD, IMAP health, folder→entity assignments, quick sync + chunk sync + wipe, classification rules when used — `mail-admin-tab.js`, `mailbox-sync-controls.js`

**UI components (inbox)**
- `messages-page.js`, `messages-focus-inbox-row.js`, `messages-conversation-sidebar.js`, `email-conversation-thread.js`, `skeleton.js` (inbox skeleton exports)

---

### 5. Marketing & YouTube — `/marketing`

YouTube channel analytics and content strategy, backed by direct API sync and an AI chat assistant.

**YouTube Data API sync**
- Syncs channel videos, metadata (title, description, published date, tags), and performance metrics
- `GET /dls/v1/youtube/status` — last sync time, video count
- `POST /dls/v1/youtube/sync` — triggers a full sync

**YouTube Analytics API**
- OAuth 2.0 flow (token stored in WP options)
- Syncs views, watch time, impressions, CTR, average view duration per video
- Data stored in custom `dls_youtube_*` tables

**AI Marketing Chat (Claude)**
- Agent: `marketing` — Senior YouTube strategist persona
- Context injection: live YouTube dashboard summary auto-prepended to system prompt
- Tool use (Anthropic Pattern C): model can call server-side tools that query `dls_youtube_*` / analytics chunks; server runs the tool loop and returns the final reply
- Session persistence: chat history stored in `dls_ai_chat_session` / `dls_ai_chat_message`
- Configured under Verwaltung → Marketing (API key, model ID)

---

### 6. AI Assistant — Stella

A contextual AI assistant built on the Anthropic Claude API, available throughout the dashboard with domain-specific agents per module.

**Agents** (defined in `inc/services/ai-agent-registry.php`):

| Agent slug | Label | Use case |
|---|---|---|
| `general` | Stella | Central assistant, all modules |
| `accounting` | Buchhaltung | Accounting rules, tax, categories, VAT |
| `clients` | Kunden | Client comms, portal, documents, referrals |
| `projects` | Projekte | Project planning, task structures, time estimates |
| `messages` | Nachrichten | Email drafting, IMAP setup, spam management |
| `marketing` | Marketing | YouTube strategy, analytics, content planning |
| `code` | Code | ddashboard architecture, code review, debugging |

**Infrastructure**
- REST: `POST /dls/v1/ai/chat/message` with `{ agent, message, session_id? }`
- Session and message history stored in custom tables (`install-ai-chat-tables.php`)
- Sessions scoped per user + agent key
- API key and model configurable per install (WP options: `anthropic_api_key`, `anthropic_model`)
- `AiChatPanel` React component embeddable anywhere: `<AiChatPanel agentKey="general" />`

---

### 7. Verwaltung (Settings & Integrations) — `/verwaltung`

Admin settings page with tabs for each integration area.

**Tabs** (`dls_verwaltung_subroutes()`)
- **Buchhaltung**: KSeF connection (NIP, token, test mode, seller identity), Saldeo API credentials
- **Projekte**: TrackingTime Pro API key
- **Marketing**: Claude API key + model ID, YouTube OAuth
- **AI-Anbindungen**: AI connectors / layers (e.g. Anthropic, Stella index URL + key, embed toggles)
- **AI-Analyse**: E-Mail-AI-Analysen (writing style + classification jobs)
- **Nachrichten**: Mailbox CRUD (IMAP settings, folder-client mapping, sync buttons, clear-client-links)

**Integration cards** display connection status, last sync time, and action buttons.

---

## Virtual App Routes

All routes are single-page React apps rendered into a WordPress page shell — no WP page posts exist for these slugs:

| Path | Module |
|---|---|
| `/` | Accounting dashboard |
| `/projects` | Project management board |
| `/marketing` | YouTube + AI marketing chat |
| `/nachrichten` | IMAP email inbox |
| `/verwaltung` | Integration settings (sub-routes: `buchhaltung`, `projekte`, `marketing`, `ai-anbindungen`, `ai-profiles`, `nachrichten`) |

---

## REST API

All browser-facing routes use the `dls/v1` namespace and require `is_user_logged_in()`.

The `api/v1` namespace is IP-restricted to two addresses (`194.126.177.181`, `23.88.90.12`) and serves the external customer portal (password reset flow, customer data, file acceptance).

**Key route groups:**

| Prefix | Purpose |
|---|---|
| `/dls/v1/nbp-rate` | EUR/PLN exchange rates |
| `/dls/v1/nbp-expense-rates` | EUR/PLN + USD/PLN + USD→EUR for expenses |
| `/dls/v1/ksef-import` | Polish e-invoice import |
| `/dls/v1/ksef-send/{id}` | Send invoice to KSeF (FA(3) XML) |
| `/dls/v1/ksef-send-status/{id}` | Check KSeF send processing status |
| `/dls/v1/saldeo-export` | Saldeo CSV export |
| `/dls/v1/subscription-billing-run` | Manual trigger for subscription billing cron |
| `/dls/v1/invoice-pdf/{id}` | Generate invoice PDF |
| `/dls/v1/invoice-download` | Download invoice PDF |
| `/dls/v1/file-pdf/{id}` | Generate file/proposal PDF |
| `/dls/v1/referral-commission-report/{id}` | Generate commission report PDF |
| `/dls/v1/client-portal-user` | Create / delete customer portal user |
| `/dls/v1/send-email` | Send email via SMTP |
| `/dls/v1/mailboxes` | Mailbox CRUD |
| `/dls/v1/emails` | Email list, fetch, update, attachment download |
| `/dls/v1/emails/clear-client-links` | Bulk-remove client_id from emails |
| `/dls/v1/pm/*` | Project management CRUD |
| `/dls/v1/ai/chat/message` | AI chat message |
| `/dls/v1/ai/chat/sessions` | Chat session list |
| `/dls/v1/ai/config` | Claude API key + model |
| `/dls/v1/youtube/sync` | YouTube data sync |
| `/dls/v1/youtube/status` | YouTube sync status |

---

## WP-Cron Jobs

| Hook | Schedule | Action |
|---|---|---|
| `dls_mail_imap_sync_cron` | Every 5 minutes | Sync all active mailboxes via IMAP (up to 50 emails each) |
| `dls_email_embed_process_cron` | Every 2 minutes | Drain `dls_email_embed_queue` → Stella `POST /emails/upsert` (when email embed is enabled) |

**Subscription billing:** There is **no** scheduled subscription billing hook. `inc/subscription-billing.php` clears any legacy `dls_subscription_billing_cron`. Billing runs only when triggered manually via **`POST /dls/v1/subscription-billing-run`** (Buchhaltung → Aktionen). `SubscriptionBillingService::run()` still performs the merge/invoices/transactions when invoked.

---

## Custom Post Types

| Slug | Purpose |
|---|---|
| `client` | Client CRM records |
| `person` | Contact persons linked to clients |
| `file` | Documents, proposals, contracts (with TipTap editor + PDF) |
| `transaction` | Income and expense entries |
| `invoice` | Invoices linked to clients and transactions |
| `product` | Product catalogue (including recurring subscription products) |
| `accountingconfig` | Per-year accounting configuration |
| `credentials` | Stored credentials (e.g. for integrations) |
| `project` | Legacy (backend only; JS PM board uses custom MySQL tables) |

---

## Custom MySQL Tables

| Table group | Tables | Purpose |
|---|---|---|
| IMAP | `dls_mailbox`, `dls_email`, `dls_email_spam_blocklist`, `dls_email_spam_whitelist`, `dls_mailbox_folder_client`, `dls_email_attachment`, `dls_email_embed_queue` | Email sync and inbox; optional Stella embed queue |
| PM | `dls_pm_project`, `dls_pm_task_list`, `dls_pm_task`, `dls_pm_task_assignee`, `dls_pm_comment`, `dls_pm_time_entry` | Project management |
| AI chat | `dls_ai_chat_session`, `dls_ai_chat_message` | AI assistant sessions and history |
| YouTube | `dls_youtube_*` (video data + analytics) | YouTube channel metrics |

---

## Security & Access Control

- All `dls/v1` routes require an authenticated WordPress session
- `api/v1` routes are IP-restricted at the `.htaccess` level
- Customer portal users (`customer` role) are blocked from all standard WordPress notification emails (new-user, password-change, lost-password)
- Programmatic password reset keys are allowed; the wp-login "lost password" flow is blocked for customers (portal handles it)
- IMAP credentials encrypted at rest (AES via `MailCrypto`)
- Email attachments stored outside the webroot (`wp-content/private/`)
- HTML email bodies rendered in sandboxed iframes (XSS isolation)

---

## PDF Generation

Two document types are generated as PDFs:

1. **Invoice PDFs** — edge-to-edge dark header with customer info, profile, Dominik info, and invoice metadata; line-item tables; generated to disk and versioned
2. **File / proposal PDFs** — TipTap or Gutenberg HTML rendered via Html2Pdf; template-driven or custom body; versioned on disk with `public_url` updated
3. **Commission report PDFs** — multi-client report with page numbering; persistent storage under `wp-content/uploads/commission-reports/` using SHA-256 filenames (never overwritten)

All PDFs use a shared CSS base and header template from `FileService`.

---

## Subscription Billing

`SubscriptionBillingService::run()` processes all active recurring products automatically:

- Groups billing lines by client and calendar month (`run-YYYY-MM` group key)
- Applies VAT rules: non-Polish billing country + real VAT number → Reverse Charge (0%); otherwise Polish default (23%)
- Generates one invoice per group with all positions; creates one income transaction (sum of net)
- Sets `service_start_date` / `service_end_date` from the min/max period of merged lines
- Transaction notes formatted as `#n title period` or `#n period: name1, name2`

---

## Design System

- Dark theme throughout; base font-size 14px (1rem floor for all text)
- SCSS design tokens in `assets/scss/variables.scss`
- Components: cards (`.card`), entity panels (`.entity-panel`), sidebar forms (`.sidebar-form`), buttons (`.button`, `.button-purple`, `.button-dark`, `.button-small`), toasts, modals, toggles, dropdowns, searchable selects, skeleton loaders
- All styles authored in `assets/scss/` — compiled CSS under `assets/css/` is never edited directly
