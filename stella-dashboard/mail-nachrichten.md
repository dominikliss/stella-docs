# Nachrichten (IMAP) — theme implementation

This document describes the **implemented** mail module in the ddashboard theme (not the separate AI-email roadmap in `.cursor/rules/ai-email-project-assistant.mdc`).

## Routes (virtual, no WP page)

| Path | UI |
|------|-----|
| `/nachrichten/` | Inbox / conversation list (`messages-page.js`) |
| `/nachrichten/verwaltung/` | Mailbox admin (IMAP credentials, sync) |

Registered in `inc/app-routes.php`. React mounts into `.dls-nachrichten`.

## Database

Option: `dls_mail_db_version` — current schema version constant: **`DLS_MAIL_DB_VERSION`** in `inc/install-mail-tables.php` (migrations run on `init` via `dls_install_mail_tables`).

| Table | Purpose |
|-------|---------|
| `{prefix}dls_mailbox` | IMAP accounts, encrypted password, folder names, optional `imap_sync_extra_folders` (chunk sync of non-inbox folders) |
| `{prefix}dls_email` | Messages per mailbox; unique `(mailbox_id, message_id, imap_folder)`; `spam_status` 0 = clean, 1 = likely spam, 2 = confirmed spam; `direction` inbound/outbound |
| `{prefix}dls_email_spam_blocklist` | Normalized sender addresses → forced spam (2) on sync |
| `{prefix}dls_email_spam_whitelist` | Normalized sender addresses → no heuristic spam (0); emoji/subdomain rules skipped |
| `{prefix}dls_mailbox_folder_client` | Optional IMAP folder → `client_id` mapping per mailbox |

## PHP services (namespace `DLS\Services` unless noted)

| File | Role |
|------|------|
| `inc/services/mailbox-db-service.php` | CRUD mailboxes/emails; `upsert_email_row` applies spam resolution; blocklist/whitelist helpers |
| `inc/services/client-email-match-service.php` | Maps addresses / domains to `client` (ACF emails, people, **website** host); subdomain detection for spam heuristic |
| `inc/services/mail-sync-service.php` | Full / incremental IMAP import |
| `inc/services/mail-chunk-sync-service.php` | Background chunked sync; optional extra folders controlled by mailbox flag |
| `inc/services/mail-thread-id-service.php` | Thread IDs |
| `inc/services/mail-crypto.php` | Mailbox password encryption |
| `inc/services/dls-imap-factory.php` | IMAP client construction |

## REST API (`dls/v1`, logged-in)

Defined in `inc/routes/mailboxes.php` (and related).

**Mailboxes:** `GET/PUT/DELETE /dls/v1/mailboxes`, `GET/POST/DELETE …/folder-clients`, `GET …/email-folders` (distinct `imap_folder` values from imported emails + counts), `GET /dls/v1/folder-clients-overview` (all mailboxes: union of imported folder paths + saved mappings, with counts and `client_id`), etc.

**Verwaltung UI:** Tabelle „IMAP-Ordner → Kunde“ unter `/nachrichten/verwaltung/` — Kundenwahl speichert per `POST`/`DELETE` auf `…/folder-clients` ohne separaten Speichern-Button.

**Emails:** `GET /dls/v1/emails` (filters: `mailbox_id`, `direction`, `thread_id`, `spam_status`, `search`, pagination).

**Metadata rebuild (batch):** `POST /dls/v1/emails/recompute-metadata` — JSON `mailbox_id` (optional; omit = all mailboxes), `threads` / `clients` (optional booleans, default `true`). Recomputes `thread_id` (see `MailThreadIdService`) and/or `client_id` from current matching rules. **Verwaltung:** button „Neu gruppieren“ per mailbox.

**Spam:** `PUT /dls/v1/emails/{id}` (`spam_status`, `client_id`), `POST …/confirm-spam`, `POST …/whitelist-sender`, `GET/DELETE …/emails/spam-blocklist`, `GET/DELETE …/emails/spam-whitelist`.

## Thread / conversation id (`thread_id`)

- Prefer **References** / **In-Reply-To** Message-IDs (stable `hdr-…` ids).
- If those are missing, a **heuristic** `heu-…` id is used: normalized subject (reply prefixes stripped) + external participant addresses (mailbox login and **owner/internal** addresses excluded — see below).
- Standalone **Message-ID** alone is **not** used as a per-message thread root (so messages without headers still group by subject/participants).

## Client assignment order (`MailSyncService`)

Applied on IMAP import and on `POST /dls/v1/emails/recompute-metadata` (`clients: true`):

1. **`thread_id`** is set first (import only; recompute uses stored `thread_id`).
2. **Folder → client** (`dls_mailbox_folder_client`): matching IMAP path sets `client_id` (**highest priority** vs address rules).
3. **Address / domain / subject** (`ClientEmailMatchService`) only if `client_id` is still empty.
4. **Outbound thread inherit**: for `direction = outbound`, if the same `thread_id` has inbound row(s) with `client_id`, the **dominant** inbound client (most frequent, then latest) is applied — **overrides** folder + address (so Gesendet follows the conversation’s Kunde).

**Verwaltung:** `GET /dls/v1/mailboxes/{id}/email-folders` returns distinct `imap_folder` values from imported emails (with inbound/total counts) to pick paths for folder mapping.

## Client auto-match (`ClientEmailMatchService`)

- Ignores **owner / house** addresses when resolving: `liss.dominikliss@gmail.com`, `liss.dominik@gmail.com`, any `@*.dominikliss.com` / `@dominikliss.com`, any `@*.foxcraft.digital` / `@foxcraft.digital` (see `is_internal_owner_email_normalized` / `is_internal_owner_domain_host`). Domain tokens in the subject matching those hosts are skipped too.
- Client **website** (`ACF website`) indexes the normalized host **and parent suffixes** (e.g. `mail.example.com` and `example.com`) so an address `@example.com` still matches when the stored URL is only a subdomain. Bare compound suffixes like `co.uk` are not registered as keys.

## Spam logic (inbound, on `upsert_email_row`)

1. **Blocklist** (`dls_email_spam_blocklist`) → `spam_status = 2` (strongest).
2. Else **whitelist** (`dls_email_spam_whitelist`) → `spam_status = 0` (heuristics skipped).
3. Else **heuristics** (raise to at least 1 if applicable):
   - **Emoji** in subject (`\p{Extended_Pictographic}`).
   - **Subdomain** sender host (heuristic + compound TLD list), except if host contains `dominikliss` or `foxcraft`, or matches a **client website** domain via `ClientEmailMatchService::host_matches_client_website()`.

Confirming spam adds addresses to the blocklist and removes them from the whitelist. Whitelisting an inbound sender adds to the whitelist, clears blocklist entry for that address, and sets all inbound rows from that `from_email` to 0.

## Frontend

| File | Notes |
|------|--------|
| `src/components/messages-page.js` | Inbox, filters, detail sidebar, spam actions, block/allow lists |
| `src/components/list-search-field.js` | Debounced search; uses `.field-container` + form input styles |

**List search:** `ListSearchField` — wrapper `field-container dls-list-search-field`; `input[type="search"]` styled in `form.scss`.

**Filter bar:** `.dls-messages-inbox .filter-bar` uses `align-items: center` (see `assets/scss/components/messages.scss` + `filter-bar.scss`).

## Build / SCSS

- JS: `npm run build` → `build/index.js`.
- SCSS: compiled via theme helper (`stella_compile_all_new_scss` in `functions.php`) into `assets/css/styles.css` (and login).

## Related Cursor rules

- **Architecture:** `.cursor/rules/architecture.mdc` — short Nachrichten summary.
- **Mail detail (optional):** `.cursor/rules/mail-nachrichten.mdc` — agent checklist for mail changes.
- **AI email roadmap (future Ollama/Chroma):** `.cursor/rules/ai-email-project-assistant.mdc` — separate from this IMAP implementation.
