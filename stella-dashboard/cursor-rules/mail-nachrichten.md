# Cursor rule export: `mail-nachrichten.mdc`

_Source of agent rule in theme repo: `.cursor/rules/mail-nachrichten.mdc`._

---

# Nachrichten (IMAP) — agent checklist

## DB tables (`dls_mail_db_version = 31`)

| Table | Purpose |
|---|---|
| `dls_mail_account` | IMAP accounts (encrypted password via `MailCrypto`) |
| `dls_mail_folder` | Per-account folder rows: `uid_validity`, `category`, `last_synced_at` |
| `dls_mail_message` | One row per `(account_id, folder_id, imap_uid)`; `direction` inbound/outbound; `thread_id`; classification: `email_category`, `email_action`, `email_urgency`, `classification_source`; `has_attachment`, `is_seen`, `is_flagged`, `subject_normalized` |
| `dls_mail_folder_link` | Polymorphic folder → entity mapping |
| `dls_mail_message_link` | Polymorphic message → entity; `source`: `folder` / `manual` / `address` |
| `dls_mail_attachment` | Attachment metadata; files under `wp-content/private/dls-mail-attachments/{account_id}/{message_id}/` |
| `dls_mail_classification_rule` | Ordered rules for Layer-1 classification |
| `dls_mail_sync_run` | Chunk sync audit log |

**Note:** Legacy tables (`dls_mailbox`, `dls_email`, `dls_email_spam_blocklist`, `dls_email_spam_whitelist`, `dls_mailbox_folder_client`, `dls_email_attachment`, embed queue) were dropped in the v3 migration. Spam is handled at the IMAP level.

## Services (`inc/services/`, namespace `DLS\Services\`)

| File | Key purpose |
|---|---|
| `mail-db-service.php` — `MailDbService` | Single DB façade: account/folder/message/attachment CRUD, client email map from WP CPTs, `list_emails` with pagination + filters, `delete_emails_for_mailbox` + manual-link transient |
| `mail-sync-v2.php` — `MailSyncV2` | IMAP sync engine: quick sync (`sync_account`) per-folder UID diff; chunk wipe-resync (`start_chunk_job` / `process_chunk_step` / `cancel_chunk_job`); `recompute_thread_ids`; `recompute_client_assignments` |
| `mail-sync-run-db-service.php` — `MailSyncRunDbService` | Append-only `dls_mail_sync_run` log; `insert_chunk_running` → `finalize_chunk_run` / `mark_cancelled_for_running_account` |
| `mail-attachment-service.php` — `MailAttachmentService` | Download IMAP attachments to private storage; 25 MB cap; MIME blocklist; `purge_attachments_for_account` |
| `mail-classification-service.php` — `MailClassificationService` | Gated by `DLS_MAIL_CLASSIFICATION_ENABLED` (default false); when on: Layer-1 rules + Layer-2 Ollama; **main INBOX inbound only**; writes `email_category`, `email_action`, `email_urgency`, `classification_source` |
| `mail-crypto.php` — `MailCrypto` | AES encrypt/decrypt for IMAP passwords |
| `dls-imap-factory.php` — `DlsImapFactory` | Webklex PHPIMAP client factory (no `ext-imap`) |

## WP-Cron

- `inc/mail-sync-cron.php` — hook `dls_mail_imap_sync_cron`, custom interval `dls_every_five_minutes` (300 s).
- Calls `MailSyncV2::sync_account($account_id, $limit)` for each active account.
- Default limit 100; filterable via `dls_mail_cron_sync_limit` (min 10).

## Quick sync vs chunk sync

- **Quick sync** (`MailSyncV2::sync_account`): per-folder UID diff (IMAP vs DB), handles **UIDVALIDITY** change (purge + reimport), flag sync up to `FLAG_SYNC_LIMIT` (500) existing UIDs, imports new UIDs up to `$limit`. Used for cron + `POST /mailboxes/{id}/sync`.
- **Chunk sync** (full wipe-resync): `start_chunk_job` wipes mailbox messages, saves manual links to transient, builds per-folder UID queues, logs run in `MailSyncRunDbService`. `process_chunk_step` imports one folder's batch. On completion: `restore_manual_links_after_resync`. Job in transient `dls_mail_sync_job_{id}` (3 h TTL). Default `uid_cap = 0` (unlimited).

## Direction detection (`MailSyncV2::extract_message_row`)

- `outbound`: IMAP path matches Sent folder pattern (`sent` / `gesendete`) **or** From address matches `DLS_OWNER_EMAILS` / `DLS_OWNER_DOMAINS` constants.
- All other messages: `inbound`.
- **`DLS_MAIL_CLASSIFICATION_ENABLED`** — default `false`; when `true`, main INBOX + inbound only (see `MailClassificationService::is_main_inbox_message_row`).

## Classification (`MailClassificationService`)

1. **Layer 1 (rules):** Ordered rules in `dls_mail_classification_rule` — first match wins; `classification_source = 'rule'`.
2. **Layer 2 (Ollama):** Only if no rule matches and Ollama URL is configured; synchronous `/api/chat` with `format: json`; `classification_source = 'ai'`.

When enabled, runs **synchronously** inside `import_uid_batch` for qualifying messages only — not queued.

## Client assignment

Links stored in `dls_mail_message_link` with `source`:

1. **`folder`**: folder → entity from `dls_mail_folder_link` — highest priority.
2. **`address`**: From/To matching against `MailDbService::build_client_email_map` (WP CPTs).
3. **`manual`**: Set by user via UI (`PUT /emails/{id}` with `client_id`). Survives wipe-resync via transient restore.

`recompute_client_assignments` deletes `folder` + `address` links and re-derives them; manual links are not touched.

## REST endpoints (all `dls/v1`)

**Mailboxes** — `dls_mail_permission`:

| Method + path | Action |
|---|---|
| GET/POST `/mailboxes` | List / create |
| GET/PUT/DELETE `/mailboxes/{id}` | Get / update / delete |
| POST `/mailboxes/test-imap` | Test IMAP credentials (no persist) |
| GET `/mailboxes/imap-health` | Health check all active accounts |
| POST `/mailboxes/list-folders` | List IMAP folder tree |
| GET `/mailboxes/{id}/email-folders` | Distinct folder paths from imported messages |
| GET/POST/DELETE `/mailboxes/{id}/folder-clients` | Folder→entity assignment CRUD |
| GET `/folder-clients-overview` | All mailboxes + folders overview |

**Sync** — `dls_mail_permission`:

| Method + path | Action |
|---|---|
| POST `/mailboxes/{id}/sync` | Quick sync (body `{ limit }`, default 100) |
| POST `/mailboxes/{id}/sync-chunk/start` | Start chunk wipe-resync (`chunk_size`, `uid_cap`) |
| POST `/mailboxes/{id}/sync-chunk/step` | Advance one chunk step |
| GET `/mailboxes/{id}/sync-chunk` | Chunk sync status |
| DELETE `/mailboxes/{id}/sync-chunk` | Cancel chunk sync |
| DELETE `/mailboxes/{id}/emails` | Wipe all messages for account |
| POST `/emails/recompute-metadata` | Recompute thread_id and/or client links (body: `{ mailbox_id?, threads?, clients? }`) |
| POST `/emails/clear-client-links` | Remove `address`-source message links (body: `{ mailbox_id? }`) |

**Emails** — `dls_mail_permission`:

| Method + path | Action |
|---|---|
| GET `/emails` | Paginated list (filters: `mailbox_id`, `client_id`, `direction`, `thread_id`, `imap_folder`, `search`, `page`, `per_page`) |
| GET/PUT `/emails/{id}` | Get / update (`client_id`, `is_seen`, `is_flagged`) |

**Classification rules** — `dls_require_login`:

| Method + path | Action |
|---|---|
| GET `/mail/classification-rules` | List rules + taxonomy metadata |
| POST `/mail/classification-rules` | Create rule |
| PUT `/mail/classification-rules/{id}` | Update rule |
| DELETE `/mail/classification-rules/{id}` | Delete rule |
| POST `/mail/classification-rules/reorder` | Reorder (body: `{ ordered_ids }`) |
| POST `/mail/messages/{id}/classify` | Re-classify single message |

## UI components

| Component | Role |
|---|---|
| `messages-page.js` | Root `/nachrichten`: `PageLayout` + dark inbox card, thread grouping by `thread_id`, tabs Kunden/WordPress/Sonstiges via `classifyThread`, filters (mailbox, direction, search), pagination, `MessagesFocusInboxRow`, `MessagesConversationSidebar`; mounts on `.dls-nachrichten` |
| `mail-admin-tab.js` | Mailbox CRUD, IMAP health, folder→entity assignments, chunk sync + wipe-resync, `MailAdminClassificationRules` (rule CRUD + reorder) |
| `mailbox-sync-controls.js` | `useMailboxSync` hook + `SyncConfirmModals`; quick sync, chunk sync (poll `step`), wipe (DELETE + start chunk), resume active jobs on mount |
| `email-classification-labels.js` | English token → German label maps; `emailClassificationDisplayDe(token, kind)`; category color tokens |
| `mail-admin-shared.js` | `mailPath()`, `mailboxSchema` (Yup), `clientLabel()` |
| `mail-conversation-resolution.js` | Thread helpers: `resolveThreadContactPerson`, `conversationListAvatarInfo`, `threadSubjectPreview`, internal-email filtering, reply builder |

**SCSS:** `messages-page.scss` (`.dls-messages-page` shell), `mail-admin-panel.scss`, `inbox.scss`, `chunk-sync.scss`, `filter-bar.scss`, `conversation-layout.scss` — imported by `messages.scss`.

## Do not

- Reference old tables (`dls_mailbox`, `dls_email`, `dls_email_spam_blocklist`, etc.) — they no longer exist.
- Use `api/v1` from browser JS for mail (use `dls/v1`).
- Skip `npm run build` after changes to `src/`.
- Edit `assets/css/*.css` directly — SCSS only.

## Docs

Full reference: `docs/stella-docs/stella-dashboard/mail-nachrichten.md`.
