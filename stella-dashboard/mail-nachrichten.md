# Nachrichten (IMAP) — theme implementation

This document describes the **implemented** mail module (v3) in the ddashboard theme.

## Routes (virtual, no WP page)

| Path | UI |
|------|-----|
| `/nachrichten/` | Inbox (`messages-page.js`) — `PageLayout` + **`.dls-messages-page__main`** (max-width column), dark **`.dls-messages-inbox-card`**: title row (Posteingang + badge), **`.client-data` + `.tabs`** (same structure as client card: Kunden / WordPress / Sonstiges), thread list **`MessagesFocusInboxRow`**, **`TablePagination`**. Loading: **`SkeletonInboxTabsStrip`**, **`SkeletonInboxFocusRows`**, **`SkeletonInboxPaginationBar`** (`skeleton.js`). Opening a thread: wide **`SidebarForm`** **`messages-conversation-sidebar.js`** — header: sender + client badge, subject below sender (like list), **`ScrollPanel`** + **`EmailConversationThread`**, mailto reply draft (no client picker / overflow menu in sidebar). List fetch: **`GET /dls/v1/emails?imap_folder=INBOX&page&per_page`** only (no mailbox/direction/search filters in UI). |
| `/nachrichten/verwaltung/` | Mailbox admin (`mail-admin-tab.js`) — IMAP credentials, sync, classification rules |

Registered in `inc/app-routes.php`. React mounts into `.dls-nachrichten`.

## Database (`dls_mail_db_version = 32`)

Option: `dls_mail_db_version` — schema version constant `DLS_MAIL_DB_VERSION` in `inc/install-mail-tables.php` (migrations run on `init` via `dls_install_mail_tables`).

**Legacy tables dropped in v3:** `dls_mailbox`, `dls_email`, `dls_email_spam_blocklist`, `dls_email_spam_whitelist`, `dls_mailbox_folder_client`, `dls_email_attachment`, embed queue. Spam is now handled at the IMAP level.

| Table | Purpose |
|-------|---------|
| `dls_mail_account` | IMAP accounts — credentials (`MailCrypto` AES), `is_active` flag |
| `dls_mail_folder` | Per-account folder rows: `uid_validity`, `category`, `last_synced_at` |
| `dls_mail_message` | One row per `(account_id, folder_id, imap_uid)`; `direction` inbound/outbound; `thread_id`; classification: `email_category`, `email_action`, `email_urgency`, `classification_source` (`rule`/`ai`); `has_attachment`, `is_seen`, `is_flagged`, `subject_normalized` |
| `dls_mail_folder_link` | Polymorphic folder → entity mapping |
| `dls_mail_message_link` | Polymorphic message → entity; `source`: `folder` / `manual` / `address` |
| `dls_mail_attachment` | Attachment metadata; files under `wp-content/private/dls-mail-attachments/{account_id}/{message_id}/` |
| `dls_mail_classification_rule` | Ordered Layer-1 classification rules |
| `dls_mail_sync_run` | Chunk sync audit log — `status` running → done/error/cancelled; counts, `errors_json`, timestamps |

## PHP services (namespace `DLS\Services\`)

| File | Role |
|------|------|
| `inc/services/mail-db-service.php` — `MailDbService` | Single DB façade: account/folder/message/attachment CRUD, client email map from WP CPTs, `list_emails` with pagination + filters, `delete_emails_for_mailbox` + manual-link transient, `restore_manual_links_after_resync` |
| `inc/services/mail-sync-v2.php` — `MailSyncV2` | IMAP sync engine: `sync_account` (quick), `start_chunk_job` / `process_chunk_step` / `cancel_chunk_job` (chunk wipe-resync), `recompute_thread_ids`, `recompute_client_assignments` |
| `inc/services/mail-sync-run-db-service.php` — `MailSyncRunDbService` | Append-only `dls_mail_sync_run` audit log; `insert_chunk_running` → `finalize_chunk_run` / `mark_cancelled_for_running_account` |
| `inc/services/mail-attachment-service.php` — `MailAttachmentService` | Download IMAP attachments to `wp-content/private/dls-mail-attachments/`; 25 MB cap; MIME blocklist; `purge_attachments_for_account`, `delete_account_attachment_tree` |
| `inc/services/mail-classification-service.php` — `MailClassificationService` | Layer-1 rule-based classification (`dls_mail_classification_rule`); Layer-2 Ollama (synchronous `/api/chat`, `format: json`) if no rule matches and Ollama URL configured; writes `email_category`, `email_action`, `email_urgency`, `classification_source`; rule CRUD + `reorder_rules` |
| `inc/services/mail-crypto.php` — `MailCrypto` | `encrypt/decrypt` for IMAP passwords (AES) |
| `inc/services/dls-imap-factory.php` — `DlsImapFactory` | `make_client()` — Webklex PHPIMAP, no `ext-imap` |

## WP-Cron

- `inc/mail-sync-cron.php` — hook `dls_mail_imap_sync_cron`, custom interval `dls_every_five_minutes` (300 s).
- Loops active accounts from `MailDbService::list_mailboxes()`, calls `MailSyncV2::sync_account($id, $limit)`.
- Default limit 100; filterable via `dls_mail_cron_sync_limit` (min 10).

## Quick sync vs chunk sync

### Quick sync (`MailSyncV2::sync_account`)
Per-folder UID diff (IMAP vs DB):
- Handles **UIDVALIDITY** change: purge all messages for that folder, reimport.
- Flag sync for up to `FLAG_SYNC_LIMIT` (500) existing UIDs — updates `is_seen`, `is_flagged`.
- Imports new UIDs up to `$limit` total across all folders.
- Called by WP-Cron and `POST /mailboxes/{id}/sync`.

### Chunk sync (full wipe-resync)
1. `start_chunk_job` — wipes all messages for the mailbox (`delete_emails_for_mailbox`), saves manual links to transient `dls_mail_pending_manual_links_{id}` (3 h), builds per-folder UID queues, logs run in `MailSyncRunDbService`.
2. `process_chunk_step` — imports one folder's queued batch; runs `MailAttachmentService` per message; runs `MailClassificationService` only when `DLS_MAIL_CLASSIFICATION_ENABLED` is true (main INBOX inbound). Returns job summary (`done`, `processed`, `queued_done`, `total`, …).
3. On completion: `restore_manual_links_after_resync` reads the transient, re-inserts manual links.
4. Job state in transient `dls_mail_sync_job_{id}` (3 h TTL). Default `uid_cap = 0` (unlimited UIDs across folders).

## Direction detection (`MailSyncV2::extract_message_row`)

- `outbound`: IMAP path matches Sent folder pattern (`sent` / `gesendete` case-insensitive) **or** From header matches `DLS_OWNER_EMAILS` or a domain in `DLS_OWNER_DOMAINS` (see `inc/constants.php`).
- All other messages: `inbound`.
- When **`DLS_MAIL_CLASSIFICATION_ENABLED`** is `true` in `inc/constants.php`, **main INBOX** + **`direction = inbound`** messages may be classified on import; outbound and IMAP subfolders are never classified (already filed). Default is **`false`** until the pipeline is ready.

## Classification (`MailClassificationService`)

**Master switch:** `DLS_MAIL_CLASSIFICATION_ENABLED` (`inc/constants.php`). When `false`, `classify_message` / REST on-demand classify return immediately (no rules, no Ollama).

When **enabled**, applied **only** to **main INBOX** (`imap_folder` exactly `INBOX`, case-insensitive) **inbound** rows (same guard in sync and REST):

1. **Layer 1 — rules:** Ordered rules in `dls_mail_classification_rule`; first match wins; `classification_source = 'rule'`.
2. **Layer 2 — Ollama:** Only if no rule matches and `AiRuntimeCredentials::ollama_base_url()` is set; synchronous POST to `/api/chat` with `format: json`; `classification_source = 'ai'`.

Classification is **synchronous** in the import request when the switch is on. Stored English tokens on `dls_mail_message`: `email_category`, `email_action`, `email_urgency`, `classification_source`. German UI labels from `src/helpers/email-classification-labels.js`.

## Client assignment

Client links are stored in `dls_mail_message_link` with `source` column:

| Source | Set by | Priority |
|--------|--------|----------|
| `folder` | `dls_mail_folder_link` entry matching the message's folder | Highest |
| `address` | `MailDbService::build_client_email_map()` → From/To/Cc matching against WP CPTs (client emails, people, invoice emails) | Second |
| `manual` | User via `PUT /dls/v1/emails/{id}` with `client_id` | Survives wipe-resync |

`recompute_client_assignments` deletes all `folder` + `address` links and re-derives them from current data. Manual links are never deleted by recompute.

`POST /dls/v1/emails/clear-client-links` removes all `address`-source links (optionally filtered by `mailbox_id`).

## REST API (`dls/v1`, logged-in)

### Mailboxes (`dls_mail_permission`)

| Method | Path | Action |
|--------|------|--------|
| GET, POST | `/mailboxes` | List / create |
| GET, PUT, DELETE | `/mailboxes/{id}` | Get / update / delete |
| POST | `/mailboxes/test-imap` | Test IMAP credentials (no persist) |
| GET | `/mailboxes/imap-health` | Health check all active accounts |
| POST | `/mailboxes/list-folders` | List IMAP folder tree |
| GET | `/mailboxes/{id}/email-folders` | Distinct folder paths from imported messages |
| GET, POST, DELETE | `/mailboxes/{id}/folder-clients` | Folder→entity assignment CRUD |
| GET | `/folder-clients-overview` | All mailboxes + folder paths (counts, assignments) |

### Sync (`dls_mail_permission`)

| Method | Path | Action |
|--------|------|--------|
| POST | `/mailboxes/{id}/sync` | Quick sync — body `{ limit }` (default 100) |
| POST | `/mailboxes/{id}/sync-chunk/start` | Start chunk wipe-resync — body `{ chunk_size, uid_cap }` |
| POST | `/mailboxes/{id}/sync-chunk/step` | Advance one chunk step |
| GET | `/mailboxes/{id}/sync-chunk` | Chunk sync status |
| DELETE | `/mailboxes/{id}/sync-chunk` | Cancel chunk sync |
| DELETE | `/mailboxes/{id}/emails` | Wipe all messages for account |
| POST | `/emails/recompute-metadata` | Recompute thread_id and/or client links — body `{ mailbox_id?, threads?, clients? }` |
| POST | `/emails/clear-client-links` | Remove `address`-source links — body `{ mailbox_id? }` |

### Emails (`dls_mail_permission`)

| Method | Path | Action |
|--------|------|--------|
| GET | `/emails` | Paginated list — filters: `mailbox_id`, `client_id`, `direction`, `thread_id`, `imap_folder` (matched **case-insensitively** to `dls_mail_message.imap_folder`), `search`, `page`, `per_page`; response `{ items, total }` |
| GET | `/emails/{id}` | Full message row |
| PUT | `/emails/{id}` | Update `client_id`, `is_seen`, `is_flagged` |

### Classification rules (`dls_require_login`)

| Method | Path | Action |
|--------|------|--------|
| GET | `/mail/classification-rules` | List rules + taxonomy metadata (categories, actions, urgencies, condition types) |
| POST | `/mail/classification-rules` | Create rule |
| PUT | `/mail/classification-rules/{id}` | Update rule |
| DELETE | `/mail/classification-rules/{id}` | Delete rule |
| POST | `/mail/classification-rules/reorder` | Reorder — body `{ ordered_ids: number[] }` |
| POST | `/mail/messages/{id}/classify` | Re-classify single message |

## Thread ID

- Prefer **References** / **In-Reply-To** Message-ID headers (stable `hdr-…` ids).
- Fallback: `heu-…` id — normalized subject + external participant addresses (owner/internal excluded).
- Standalone Message-ID alone is not used as a thread root.

`MailSyncV2::recompute_thread_ids` can rebuild thread IDs across all stored messages from their stored headers.

## Frontend

| File | Notes |
|------|--------|
| `src/components/messages-page.js` | Root `/nachrichten` — `PageLayout` (Aktualisieren in header only; no inbox filter button). **`useDlsQuery`** loads paged INBOX list; **`groupEmailsIntoThreads`** + tab filter (**`classifyThread`**) client-side. Tabs: **`.client-data` > `.tabs` > `.tab`** (+ **`tab-badge`** counts), same DOM pattern as `client-card.js`. **`MessagesFocusInboxRow`** (avatar, name, client badge, snippet, time). **`MessagesConversationSidebar`** + **`EmailConversationThread`**. **`skeleton.js`**: inbox card skeleton while `loading`. Opening a thread does **not** auto-mark read or refetch list. Mounts on `.dls-nachrichten` |
| `src/components/mail-admin-tab.js` | Mailbox CRUD, IMAP health, folder→entity mappings, chunk sync + wipe-resync, `MailAdminClassificationRules` — full rule CRUD (GET list + metadata, POST/PUT/DELETE rule, POST reorder) |
| `src/components/mailbox-sync-controls.js` | `useMailboxSync` hook + `SyncConfirmModals`; quick sync, chunk sync (poll `step` ~400 ms), wipe (DELETE + start chunk), resume active jobs on mount via `Promise.allSettled` |
| `src/helpers/email-classification-labels.js` | English token → German label maps for categories, actions, urgency, condition types, source; `emailClassificationDisplayDe(token, kind)`; `EMAIL_CATEGORY_COLOR` |
| `src/helpers/mail-admin-shared.js` | `mailPath(suffix)`, `mailboxSchema` (Yup), `clientLabel(client)` |
| `src/helpers/mail-conversation-resolution.js` | Thread helpers: `resolveThreadContactPerson`, `conversationListAvatarInfo`, `threadSubjectPreview`, internal-email filtering, avatar, reply `mailto:` builder |

**SCSS:** `components/messages-page.scss` (`.dls-messages-page`, inbox card, focus rows, conversation sidebar chrome), `mail-admin-panel.scss`, `inbox.scss`, `chunk-sync.scss`, `conversation-layout.scss`, `components/skeleton.scss` (incl. inbox focus-row skeleton), `client-card.scss` (shared **`.client-data .tab-badge`**), `header.scss` (global **`.tabs` / `.tab`**). `filter-bar.scss` remains in the bundle for other screens; the Nachrichten inbox no longer embeds the filter bar. Imported via `styles.scss` / `messages.scss` as before.

## Build / SCSS

- JS: `npm run build` → `build/index.js` (required after any `src/` change).
- SCSS: compiled via `stella_compile_all_new_scss` in `functions.php` into `assets/css/styles.css`. Never edit CSS directly.

## Related Cursor rules

- **Architecture:** `.cursor/rules/architecture.mdc` — Nachrichten summary + file structure.
- **Mail detail:** `.cursor/rules/mail-nachrichten.mdc` — agent checklist for mail changes.
- **AI email roadmap (future):** `.cursor/rules/ai-email-project-assistant.mdc` — separate from this IMAP implementation.
