# Cursor rule export: `mail-nachrichten.mdc`

_Source of agent rule in theme repo: `.cursor/rules/mail-nachrichten.mdc`._

---

# Nachrichten (IMAP) — agent checklist

## DB tables (`dls_mail_db_version`)

| Table | Purpose |
|---|---|
| `dls_mailbox` | IMAP account credentials (encrypted password via `MailCrypto`) |
| `dls_email` | Imported emails: `spam_status` 0/1/2, `client_id`, `direction` inbound/outbound, `thread_id`, `body_has_quote_history` (virtual) |
| `dls_email_spam_blocklist` | Blocked sender emails (normalized) |
| `dls_email_spam_whitelist` | Whitelisted sender emails |
| `dls_mailbox_folder_client` | IMAP folder → client_id mapping (highest priority for auto-assign) |
| `dls_email_attachment` | Attachment metadata; files under `wp-content/private/dls-mail-attachments/` |

## Services (all under `inc/services/`, namespace `DLS\Services\`)

| File | Key purpose |
|---|---|
| `mailbox-db-service.php` | CRUD + spam resolution + `clear_all_email_client_links(?int $mailbox_id)`; `list_email_ids_with_uid_for_folder()`; `delete_email_and_attachments()` |
| `mail-sync-service.php` | Sync single/all folders; `import_single_message`; date fallback chain; `recompute_thread_ids`, `recompute_client_assignments`; after each folder sync calls `prune_emails_removed_from_imap_folder()` |
| `mail-chunk-sync-service.php` | Background chunked sync (start/step/cancel via WP transients); on successful job completion calls `prune_imap_orphans_for_completed_chunk_job()` → same prune per folder |
| `client-email-match-service.php` | `resolve_client_id_single_counterparty_exact()` (IMAP import); `resolve_client_id()` (legacy) |
| `mail-html-split-quote-service.php` | `split_latest_and_quoted()` — removes quoted reply history for list payloads |
| `mail-attachment-storage-service.php` | Store Webklex `Attachment` bytes; `list_attachments_for_email`, `get_attachment_row`, `get_absolute_path` |
| `mail-thread-id-service.php` | `compute_thread_id()` from Message-ID / In-Reply-To / References headers |
| `mail-crypto.php` | `MailCrypto::encrypt/decrypt` for IMAP passwords |
| `dls-imap-factory.php` | `DlsImapFactory::make_client()` — Webklex PHPIMAP, no `ext-imap` |

## WP-Cron

- `inc/mail-sync-cron.php` — hook `dls_mail_imap_sync_cron`, custom interval `dls_every_five_minutes` (300 s).
- Syncs all active mailboxes; limit filterable via `dls_mail_cron_sync_limit` (default 50).

## Import limit vs. IMAP orphan prune (UID)

- **Import / fetch:** Each folder sync uses Webklex `messages()->all()->setFetchOrderDesc()->limit($limit)` — only the **newest N** messages are **fetched and upserted**. Older messages are not re-touched every run; that is intentional.
- **Prune (remove DB rows when a message vanished from IMAP):** **Independent** of that N. `MailSyncService::prune_emails_removed_from_imap_folder()` loads **all UIDs currently in the open folder** via `fetch_imap_uid_set_for_open_folder()` (connection `getUid()->data()`), compares to DB rows from `MailboxDbService::list_email_ids_with_uid_for_folder()`, and deletes missing UIDs with `delete_email_and_attachments()`.
- **Implication:** A message that stays on the server but falls **outside** the latest-N fetch batch is **not** pruned — it is still in the server UID set. You do **not** need to “count new mail” for prune correctness.
- **Rows without `imap_uid`:** Not matched by UID prune; left unchanged (see docblock on `prune_emails_removed_from_imap_folder`).
- **IMAP caveat:** **UIDVALIDITY** changes remap UIDs; treat unusual mass prune as a signal to verify server/folder state.
- **Debug:** Folder sync `debug` includes `imap_pruned` (count deleted that run).

## REST endpoints (all `dls/v1`, `dls_mail_permission`)

| Method + path | Action |
|---|---|
| GET/POST `/mailboxes` | List / create |
| GET/PUT/DELETE `/mailboxes/{id}` | Get / update / delete |
| POST `/mailboxes/test-imap` | Test IMAP credentials (no persist) |
| POST `/mailboxes/list-folders` | List IMAP folder tree |
| POST `/mailboxes/{id}/sync` | Quick sync (body `{ limit }`) |
| POST `/mailboxes/{id}/sync-chunk/start` | Start background chunk sync |
| POST `/mailboxes/{id}/sync-chunk/step` | Advance one chunk step |
| DELETE `/mailboxes/{id}/sync-chunk` | Cancel chunk sync |
| GET/POST/DELETE `/mailboxes/{id}/folder-clients` | Folder→client mapping CRUD |
| GET `/folder-clients-overview` | All mailboxes + folders overview (Verwaltung table) |
| GET `/mailboxes/{id}/email-folders` | Distinct IMAP folder paths from imported emails |
| GET `/emails` | Paginated email list (filters: mailbox_id, client_id, direction, thread_id, spam_status, search) |
| GET `/emails/{id}` | Full email row (body_html includes quoted history) |
| GET `/emails/{id}/stella-chroma-raw` | Stella `GET /emails/document/email_{id}` → `{ id, document, metadata }` im JSON (Kopfdaten) |
| PATCH `/emails/{id}` | Update client_id or spam_status |
| GET `/emails/{id}/attachments` | List attachments |
| GET `/emails/{id}/attachments/{attachment_id}` | Download attachment |
| POST `/emails/confirm-spam` | Mark email as spam + blocklist sender |
| POST `/emails/whitelist-sender` | Whitelist From + clear spam flag on thread |
| GET `/emails/spam-blocklist` | List blocklist |
| DELETE `/emails/spam-blocklist` | Remove blocklist entry (`?email=`) |
| GET `/emails/spam-whitelist` | List whitelist |
| DELETE `/emails/spam-whitelist` | Remove whitelist entry (`?email=`) |
| POST `/emails/recompute-metadata` | Recompute thread_id and/or client_id for stored emails (body: `{ mailbox_id?, threads?, clients? }`) |
| POST `/emails/clear-client-links` | Set client_id = NULL (body: `{ mailbox_id? }` — omit for all) |

## Client auto-assign rules (strict, 2025-03)

Priority order (higher wins, later steps only run if `client_id` is still null):

1. **Folder mapping:** `dls_mailbox_folder_client` → `apply_folder_client_match()`. Never overwritten by address logic.
2. **Address rule (`apply_address_client_match`):**
   - Collect **To + Cc + Bcc + From** (no Reply-To).
   - Drop: mailbox login, `dominikliss.com` + subdomains, `foxcraft.digital` + subdomains, `liss.dominik@gmail.com`, `liss.dominikliss@gmail.com`.
   - **Exactly 1** distinct external address → assign if it matches any client main email / invoice email / linked person email.
   - **More than 1** → assign only if **every** address maps to the **same** client in the email index.
   - Otherwise: no assignment.
3. No domain-from-address, no subject heuristics, no outbound thread inheritance, no folder consensus propagation.

`ClientEmailMatchService::resolve_client_id_single_counterparty_exact()` implements rules 1+2 above.  
`ClientEmailMatchService::resolve_client_id()` is legacy (domain + subject heuristics) — **not** used by import.

**Bulk clear:** `MailboxDbService::clear_all_email_client_links(?int $mailbox_id)` + `POST /dls/v1/emails/clear-client-links`. UI: "Zuordnungen löschen" in `mail-admin-tab.js` with confirm modal. Does not delete folder→client mappings.

## date_sent fallback chain

`MailSyncService::message_to_row()` resolves `date_sent` in order:

1. Webklex `getDate()` (parsed `Date:` header).
2. Raw `Date:` header scan via `extract_date_header_value_from_raw()` (handles line folding).
3. iCalendar body parse via `infer_date_sent_from_message_content()` — scans text body, HTML, `text/calendar` / `.ics` / `.vcs` attachments, and raw MIME body for `BEGIN:VCALENDAR`; reads first usable value in order DTSTAMP → CREATED → LAST-MODIFIED → DTSTART. Supports UTC (`…Z`), floating, `TZID=`, `VALUE=DATE` (all-day → midnight). Stored in site timezone (`wp_timezone()`).

## Quote splitting

List endpoint strips quoted reply history from `body_html` via `MailHtmlSplitQuoteService::split_latest_and_quoted()`. Flag `body_has_quote_history: true` on list items means a `GET /emails/{id}` fetch is needed for the full content.

## UI components

| Component | Role |
|---|---|
| `messages-page.js` | Root: `/nachrichten` — `PageLayout` + grid, dark `.card` inbox, collapsible filters, `messages-focus-inbox-row.js`, conversation detail in `messages-conversation-sidebar.js` (`SidebarForm`), Schnellzugriff + complementary column; mounts on `.dls-nachrichten` |
| `messages-focus-inbox-row.js` | Inbox row: avatar, name • thread ref, snippet, distinct `email_category` pills (German), time, Antworten, overflow links |
| `messages-conversation-sidebar.js` | Wide `SidebarForm`: thread header (no `<header>`), `ScrollPanel` + `EmailConversationThread`, Kunde, mailto reply |
| `mail-admin-tab.js` | Mailbox CRUD, IMAP test, chunk sync, folder-client mapping table, recompute metadata, clear client links |
| `master-detail-layout.js` | Two-column master/detail (`.dls-master-detail`) — still used elsewhere; Nachrichten inbox no longer uses it |
| `scroll-panel.js` | Scrollable flex child with optional header/footer |
| `conversation-list-item.js` | Legacy thread row styling (dark master-detail); not used on `/nachrichten` (uses `messages-focus-inbox-row.js`) |
| `email-conversation-thread.js` | Thread view: messages sorted asc by date, date separators |
| `email-message-block.js` | Single message bubble (header, body, attachments) |
| `email-detail-sidebar.js` | Sidebar: client assignment picker, spam/whitelist actions |
| `email-html-iframe.js` | Sandboxed iframe for HTML email bodies |
| `email-bubble-body.js` | Chooses iframe vs plain text render |
| `email-meta-table.js` | From / To / Cc metadata display |
| `message-attachment-list.js` | Attachment chip list with download |
| `message-thread.js` | Container for a list of `EmailMessageBlock`s |
| `message-date-separator.js` | Date label row between messages |
| `list-search-field.js` | Debounced search input (`.field-container` + `input[type="search"]` styles) |
| `table-pagination.js` | Server-paginated pager: prev/next chevrons flanking numeric page list + ellipsis, compact `from–to / total · current/total` meta (`DEFAULT_TABLE_PAGE_SIZE = 200`) |

**SCSS:** `messages.scss` (badges, sync bar), `conversation-layout.scss` (thread bubbles, legacy list), `messages-page.scss` (dark `/nachrichten` shell under `.dls-messages-page`), `filter-bar.scss` (filter row), `inbox.scss` (blocklist tables).

**JS helpers** (`src/helpers/`):
- `mail-admin-shared.js` — `mailPath()`, `mailboxSchema` (Yup), `clientLabel()`
- `mail-conversation-resolution.js` — `normalizeMailAddress`, `isInternalOwnerEmailNormalized`, `resolveThreadContactPerson`, `threadFallbackContactLabel`, `conversationListAvatarInfo`, `threadSubjectPreview`, `personDisplayName`
- `html-mail-plain-text.js` — plain-text extraction from email HTML

## Do not

- Use `api/v1` from browser JS for mail (use `dls/v1`).
- Skip `npm run build` after any change to `src/components/` or `src/helpers/`.
- Edit `assets/css/*.css` directly — SCSS only.
- Add domain / website / subject heuristics back to `apply_address_client_match` — they cause false positives.

## Docs

Full reference: `docs/mail-nachrichten.md`.
