# ddashboard — WordPress Theme Architecture

## Stack
- PHP (WordPress theme) + React (JS built via webpack → `build/index.js`)
- ACF Pro for all custom fields (config in `inc/acf/settings/*.json`)
- Data layer: WordPress REST API via `@wordpress/core-data` (`useEntityRecords`)
- Forms: Formik + Yup validation
- **No GraphQL** — it has been removed. Do not reference `@apollo/client`, `gql`, or `transactionFields.*` paths.

## Custom Post Types
| Post Type slug     | WP REST API post type |
|--------------------|-----------------------|
| `client`           | `client`              |
| `person`           | `person`              |
| `file`             | `file`                |
| `transaction`      | `transaction`         |
| `invoice`          | `invoice`             |
| `product`          | `product`             |
| `accountingconfig` | `accountingconfig`    |
| `credentials`      | `credentials`         |
| `project`          | `project` (backend only — JS frontend removed) |

### Product (`product`)
- ACF `client_id` — optional post object → `client` (`return_format: id`); `product-form.js` uses `SearchableSelect`; `normalizeProductAcfForRest` coerces empty/`0` to `null` for REST.

### File (`file`)
- **CPT** supports `title` + `editor` (`post_content`). Body is edited in the dashboard with **TipTap** (`src/components/tiptap-editor.js` + `file-form.js`). Legacy Gutenberg markup in the DB is loaded via REST `content.rendered` for the first edit; saves persist `content.raw` as TipTap HTML.
- **ACF** (group `group_6762f36b7113a`): `is_template` (true/false — reusable document without a client), `content_mode` (`custom` | `from_template`), `template_file_id` (post object → another `file` that has `is_template`), `document_type` (`normal` | `terms_of_service`). `client_id` / `public_url` are hidden when `is_template` is on.
- **`document_type` (React):** In `file-form.js`, the **Dokumentart** field is shown **only when `type === "document"`**. Yup requires it in that case. On save, if type is not `document`, persist **`document_type: "normal"`** (AGB vs normal only applies to real documents).
- **Template-driven title:** `TemplateMetadataSync` + `buildFileTitleFromTemplateAndClient()` set the post **title** when using a template: template title + client website domain in parentheses, e.g. `Vorlage (example.com)`. Uses `clientRecordRef` / `clientsRef` so the title updates when client data loads after template selection.
- **Edit template from file form:** When `content_mode === from_template` and a template is selected, an **edit icon** opens a **second** `SidebarForm` (`className="sidebar-form--template-edit"`, higher `z-index`) with nested `FileForm` (`databaseId` = template id, `forceIsTemplate`, `editingTemplateContext`) and a visible **“Vorlage”** banner. Invalidate `getEntityRecords` for `file` after template save (`FILE_LIST_QUERY` in `file-form.js` matches `EntityService` / `clients.js` `FILE_QUERY`).
- **Clients UI**: `clients.js` opens a wide `SidebarForm` with `FileForm`; `client-card.js` lists non-template files per client via `DocumentList` (add / edit / delete). Template files are excluded from client lists (`acf.is_template`). **Reports tab** empty state uses `DocumentList` with `emptyText` (same empty styling as Dateien), not a plain `<p>`.
- **PDF** (`inc/services/file-service.php`): `get_file_body_html_for_pdf( $file_id )` applies `the_content` to `post_content` from this file or, when `content_mode === from_template`, from `template_file_id`. `transform_file_body_html_for_pdf()` maps both Gutenberg classes (`wp-block-heading`, `wp-block-list`) and plain TipTap tags (`<h2>`, `<ul>`) into Html2Pdf markup. `create_proposal()` writes versioned PDFs (`-v1`, `-v2`, …) and updates `public_url` meta; returns early for `is_template` files (no client / no PDF). `list_pdf_versions_for_file()` globs on-disk PDFs for REST.
- **REST**: `POST /dls/v1/file-pdf/{id}` (logged-in) runs `create_proposal`; `file` responses include `dls_pdf_versions` (`[{ version, url }]`). `file-form.js` calls this after save for non-template files. Clients list uses `dls_pdf_versions` in `DocumentList` for per-version links; primary link prefers a `.pdf` `public_url`.
- After changing `inc/acf/settings/group_6762f36b7113a.json`, sync the field group in WP admin (ACF) so PHP `get_field()` and `BasicService` SQL include the new meta keys.

## PHP Namespaces
- `DLS\Services\` — all services (`FileService`, `ClientService`, `OptionService`, `BasicService`, `KSeFService`, `KSeFXmlBuilder`, `BankAccountDbService`, `SaldeoExportService`, `SubscriptionBillingService`, `YouTubeDataService`, `YouTubeSyncService`, `YouTubeOAuthService`, `YouTubeAnalyticsApiService`, `YouTubeAnalyticsSyncService`, `AiAgentRegistry`, `AnthropicChatService`, `AiChatSessionService`, `PmDbService`, `MailDbService`, `MailSyncV2`, `MailSyncRunDbService`, `MailAttachmentService`, `MailClassificationService`, `MailCrypto`, `DlsImapFactory`, `ExpenseReceiptStorageService`, `WritingStyleHistoryService`, `NbpExchangeRateService`, `InvoicePdfService`, `CommissionReportService`). Services are autoloaded via Composer classmap.
- No namespace — route callback functions (`dls_*` prefix convention)

## Subscription billing (WP-Cron)
- File: `inc/subscription-billing.php` — clears legacy `dls_subscription_billing_cron`; **no automatic runs**. Manual only: `POST /dls/v1/subscription-billing-run` (Buchhaltung).
- `SubscriptionBillingService` uses client `country` + `vat_number`: non‑Poland billing country (`country` not PL/Poland/Polska/…) + real VAT/UID → `vat_free` = `Reverse Charge`, `vat_percent` = `0`; else `nein` + **`23%`** (Poland default). Same client + same **billing group**: monthly/quarterly group by `period_end` Y‑m; yearly by `period_start` Y‑m (first month of the year period). **Quarterly** lines only become billable from the **first day of the quarter’s last month**; **yearly** from **Jan 1** of that calendar year. **Transaction `invoice_date`:** monthly = run date (today); quarterly = **last day of the quarter**; yearly = **first day of the year** (mixed yearly+quarterly or yearly+monthly on one invoice → today). Product `client_id` is normalized from ACF (post_object / ui). **One invoice** = multiple `positions`, one Einnahme (`price_net` = sum); `service_start_date` / `service_end_date` = min/max of merged lines. **Transaction `note`:** default `#<n> <title> <period>` or `#<n> <period>: name1, name2` (no `Abo` prefix). **Hosting** / **Wartung** rules unchanged.

## Account emails (`customer` role)
- File: `inc/user-account-email-block.php` (loaded from `functions.php` after `user-roles.php`). Blocks core WordPress emails for users with role **`customer`**: new-user notifications (user + admin + legacy filters), password-change / email-change notices, admin password-change notification, and the **lost-password notification email** (`send_retrieve_password_email` false). Programmatic reset keys (`get_password_reset_key` / custom REST) are allowed. The wp-login “lost password” form does not create a key or send mail for customers (use the remote portal flow). Does not affect other plugins’ mail unless they reuse these hooks.

## Client ↔ portal user (REST)
- File: `inc/client-portal-user-rest.php` — registers `dls_has_portal_user` (boolean) on `wp/v2/client` via `register_rest_field`. True when any WP user has ACF user field `client` / `field_6961c28c7bc71` (post_object → client ID) matching that client. Batched lookup per request (`dls_portal_user_client_ids_map()`); `dls_portal_user_client_ids_map_clear_cache()` after creating or deleting a portal user.
- File: `inc/routes/client-portal-user-create.php` — **`POST /dls/v1/client-portal-user`** with `{ client_id }` (logged-in). Creates WP user `customer`, username from sanitized title/official name, email from first valid `invoice_emails[].email` then `email`, ACF `client` link, strong random password, **no WP notification emails** (filters). Response includes plaintext password once + portal URL `https://dominikliss.com/kundenbereich/`. **Auto file:** If a published **`file`** template exists with **post title exactly** `Allgemeine Geschäftsbedingungen` and `is_template`, creates a client **`file`**: `type` document, `document_type` terms_of_service, `content_mode` from_template, linked `template_file_id`, `client_id` set; then runs `FileService::create_proposal()` for PDF. Response may include `agb_file_created`, `agb_file_id`. **`DELETE /dls/v1/client-portal-user?client_id=`** — deletes **only** the linked **`customer`** WP user(s); does **not** delete the client or files. `clients.js`: after POST success toast + `invalidateClients()`; if `agb_file_created`, extra toast + `invalidateFiles()`. After DELETE: toast + `invalidateClients()`.

## REST API Routes
| Namespace  | Purpose                          | Auth                        |
|------------|----------------------------------|-----------------------------|
| `api/v1`   | External app routes              | IP-restricted in `.htaccess` — **never use from browser JS** |
| `dls/v1`   | Browser-facing internal routes   | `is_user_logged_in()` check |

New browser-facing endpoints MUST use `dls/v1` namespace. The `api/v1` namespace is blocked for all IPs except `194.126.177.181` and `23.88.90.12`.

## File Structure
```
inc/
  constants.php              # App-wide constants: DLS_ALLOWED_IPS, DLS_OWNER_EMAILS, DLS_OWNER_DOMAINS, DLS_MAIL_CLASSIFICATION_ENABLED, DLS_ANALYSIS_TIMEOUT
  subscription-billing.php   # Clears legacy subscription cron; billing is manual via REST only
  mail-sync-cron.php         # WP-Cron: `dls_mail_imap_sync_cron` every 5 min → sync active mailboxes
  install-mail-tables.php    # dbDelta v3 (DLS_MAIL_DB_VERSION = 31): dls_mail_account, dls_mail_folder, dls_mail_message, dls_mail_folder_link, dls_mail_message_link, dls_mail_attachment, dls_mail_classification_rule, dls_mail_sync_run; drops legacy dls_mailbox / dls_email tables on upgrade
  install-pm-tables.php      # dbDelta: dls_pm_project, dls_pm_task_list, dls_pm_task, dls_pm_task_assignee, dls_pm_comment, dls_pm_time_entry; migrations v1–v2
  install-bank-account-tables.php # dbDelta: dls_bank_account (label, iban, bic, bank_name, account_holder, condition_country, condition_vat, currency, is_primary, US fields); migrations v1–v3; seeds defaults
  install-ai-chat-tables.php # dbDelta: dls_ai_chat_session, dls_ai_chat_message
  install-writing-style-history.php # dbDelta: dls_writing_style_run + analysis_type (v2); append-only email AI analysis history
  install-youtube-tables.php # dbDelta: YouTube Data + Analytics custom tables
  acf/settings/              # ACF field group JSON configs (group_*.json)
  post-types/                # CPT registration
  routes/                    # REST API endpoints (dls/v1 namespace unless noted)
    mailboxes.php            # Mailbox CRUD, folder-entity assignments, folder-clients-overview, IMAP health/test/list-folders
    mail-emails.php          # Email list/get/update endpoints
    mail-sync.php            # Quick sync, chunk wipe-resync (start/step/cancel/status), recompute-metadata, clear-client-links, DELETE emails
    mail-classification.php  # Classification rules CRUD + reorder; POST /mail/messages/{id}/classify
    pm-projects.php          # /dls/v1/pm/* — projects, task lists, tasks, assignees, comments, time entries
    tracking-time.php        # TrackingTime Pro import (time entries, tasks, projects)
    bank-accounts.php        # GET /dls/v1/bank-accounts, POST, PUT /{id}, DELETE /{id}
    ksef-import.php          # POST /dls/v1/ksef-import
    ksef-test-connection.php # POST /dls/v1/ksef-test-connection
    ksef-send.php            # POST /dls/v1/ksef-send/{id}, GET /dls/v1/ksef-send-status/{id}
    youtube-sync.php         # POST /dls/v1/youtube/sync, GET /dls/v1/youtube/status
    youtube-analytics.php    # OAuth + YouTube Analytics API sync → dls_youtube_analytics_*
    saldeo-export.php        # GET  /dls/v1/saldeo-export
    expense-receipt-import.php # POST /dls/v1/expense-receipt-import (multipart PDF/JPEG → Ausgabe); GET /dls/v1/expense-receipt/{transaction_id} (inline Beleg)
    subscription-billing-run.php # POST /dls/v1/subscription-billing-run
    client-portal-user-create.php # POST/DELETE /dls/v1/client-portal-user
    nbp-rate.php             # GET  /dls/v1/nbp-rate, GET /dls/v1/nbp-expense-rates (EUR/PLN + USD/PLN + USD→EUR for USD expenses)
    invoice-pdf.php          # POST /dls/v1/invoice-pdf/{id}
    invoice-download.php     # GET  /dls/v1/invoice-download
    referral-commission-report.php # GET /dls/v1/referral-commission-report/{id}
    send-email.php           # POST /dls/v1/send-email
    ai-chat.php              # POST /dls/v1/ai-chat — Claude via AnthropicChatService
    ai-writing-style.php     # Writing-style: GET profile, POST learn/stop, final-grounding; history list (analysis_type query)
    ai-email-classification-suggestions.php # Klassifizierung: GET profile, POST learn/stop/final-grounding; cron dls_email_classification_cron_run
    authenticate.php         # api/v1 (IP-restricted): POST /generate-password-key, POST /set-password — portal password reset flow
    customer-data.php        # api/v1 (IP-restricted): GET /customer-data, POST /accept-file — customer portal data + file acceptance
  services/                  # PHP service classes (DLS\Services\)
    mail-db-service.php      # MailDbService — single DB façade for accounts/folders/messages/attachments/links; client email map from WP CPTs; list_emails with pagination
    mail-sync-v2.php         # MailSyncV2 — IMAP sync engine: quick sync (sync_account), chunk wipe-resync (start/process/cancel_chunk_job), recompute_thread_ids, recompute_client_assignments
    mail-sync-run-db-service.php # MailSyncRunDbService — append-only dls_mail_sync_run log for chunk jobs
    mail-attachment-service.php  # MailAttachmentService — download IMAP attachments to private storage; 25 MB cap; MIME blocklist
    mail-classification-service.php # MailClassificationService — Layer-1 rules (dls_mail_classification_rule); Layer-2 Ollama optional; synchronous on import
    mail-crypto.php          # MailCrypto — AES encrypt/decrypt for IMAP passwords
    dls-imap-factory.php     # DlsImapFactory — Webklex PHPIMAP client factory (no ext-imap)
    pm-db-service.php        # PmDbService: projects / task lists / tasks / assignees / comments / time entries
    ksef-service.php         # KSeF API 2.0 (test + production, import + send); selects token by ksef_test_mode
    ksef-xml-builder.php     # FA(3) XML generation for outgoing invoices (schema 1-0E)
    bank-account-db-service.php # BankAccountDbService — dls_bank_account CRUD + find_account / set_primary / get_primary_for_currency
    saldeo-export-service.php
    expense-receipt-storage-service.php # private receipts under wp-content/private/dls-expense-receipts/{year}/{month}/
    subscription-billing-service.php
    nbp-exchange-rate-service.php
    option-service.php       # WP options: ksef_* (incl. test_mode, token, test_token, seller_*), saldeo_*, youtube_*, tracking_time_pro_*
    youtube-data-service.php
    youtube-sync-service.php
    youtube-oauth-service.php
    youtube-analytics-api-service.php
    youtube-analytics-sync-service.php
    ai-agent-registry.php    # AiAgentRegistry — named Claude agent configs
    anthropic-chat-service.php # AnthropicChatService — Claude API HTTP client
    ai-chat-session-service.php # AiChatSessionService — dls_ai_chat_session / dls_ai_chat_message
    ai-youtube-chat-tools.php  # YouTube-specific tool definitions for AI agent
    writing-style-history-service.php # dls_writing_style_run — analysis_type writing_style | classification_suggestions; list_runs/count_runs filter
    acf-service.php
    transaction-service.php
    project-service.php
    client-service.php
    basic-service.php
    file-service.php         # PDF generation, TipTap (core); delegates invoice PDFs and commission reports to split services
    invoice-pdf-service.php  # InvoicePdfService — invoice PDF generation; bank account selected via three-tier priority (invoice pin → transaction pin → BankAccountDbService::find_account)
    commission-report-service.php # CommissionReportService — referral commission PDFs (split from FileService)
  shortcodes/                # [shortcode] registrations
  rewrite-rules.php          # Extra add_rewrite_rule calls (client slug pattern, etc.) — separate from app-routes.php
  meta-boxes/
    file-meta-box.php        # Custom meta box for file CPT
  scripts/
    learn-writing-style.php  # CLI / cron — Schreibstil aus gesendeten Mails; dls_writing_style_* options
    learn-email-classification-suggestions.php # CLI / cron — Taxonomie aus eingegangenen Mails; dls_email_classification_* options
src/
  components/                # React components
    accounting.js            # Main dashboard — PageLayout (filter + actions), chart, transaction list, KSeF import, Beleg import modal
    expense-receipt-import-modal.js # Multipart upload → expense-receipt-import REST; year/month picker
    transactions.js          # Reusable transaction table with expand/edit/delete (uses EntityListTable)
    transaction-form.js      # Formik form for creating/editing transactions (uses useFormSubmit)
    transaction-card.js      # Expanded detail view for a single transaction (uses DataTable + CrudActions)
    sidebar-form.js          # Slide-in panel; optional `wide`, `className` (e.g. sidebar-form--template-edit)
    file-form.js             # Formik form for file CPT (TipTap, template modes)
    tiptap-editor.js         # TipTap StarterKit wrapper for file body HTML
    expanding-content.js     # Animated height expand/collapse
    invoices.js / invoice-form.js  # Invoices list uses PageLayout + EntityListTable + useEntityCrud; detail uses DataTable; bank account is ACF `bank_account_id` (select); language-based default via `helpers/invoice-bank-account.js`; choices from `acf/load_field` on invoice CPT
    bank-account-settings.js # Bank account CRUD panel (EntityListPanel + SidebarForm); side-by-side with KSeF settings in management-page
    clients.js / client-form.js / client-card.js
    products.js / product-form.js  # Products list uses EntityCardList + useEntityCrud; form uses useFormSubmit
    open-invoices.js               # Uses PageLayout + EntityListTable + useEntityCrud
    accounting-configs.js / accounting-config-form.js  # Uses EntityCardList + useEntityCrud
    accounting-stats.js
    person-form.js / person-card.js  # PersonForm uses useFormSubmit; also fixes missing disabled={isSubmitting}
    document-list.js / file-card.js
    portal-credentials-modal.js
    entity-list-panel.js     # Generic panel shell (title, action, loading spinner, empty-state, children body)
    page-layout.js           # Global page layout shell: optional tabs (.sub-header) + heading (.site-meta h1 + actions/filter) + children
    entity-card-list.js      # Generic card-style entity list (.card > .data-list) with skeleton + edit/delete actions
    entity-list-table.js     # Generic data-grid table with skeleton, empty state, and ExpandingContent detail rows
    data-table.js            # DataTable + DataTableRow — generic key-value display (.data-table > .data-table-row)
    crud-actions.js          # CrudActions — standardised icon-button action group (flex row of .button-crud)
    formik-date-picker.js    # Formik + react-datepicker wrapper; stores Y-m-d (or "" when empty); locale=de
    chat-bubble.js           # Generic chat bubble — align="start" (counterpart) or "end" (own/outbound)
    project-management.js    # PM board (projects / task lists / tasks / comments; uses useDlsQuery + useRestAction + DataTable)
    messages-page.js         # /nachrichten — PageLayout + dark inbox card, thread grouping by thread_id, tabs Kunden/WordPress/Sonstiges, filters (mailbox, direction, search), pagination, MessagesFocusInboxRow, MessagesConversationSidebar; mounts on .dls-nachrichten
    email-conversation-thread.js # Thread view (sorted messages, date separators)
    email-message-block.js   # Single email bubble (from, date, body, attachments)
    email-detail-sidebar.js  # Sidebar: client link, spam actions
    email-bubble-body.js     # Email body renderer (HTML iframe or plain text)
    email-html-iframe.js     # sandboxed iframe for HTML emails
    email-meta-table.js      # From/To/Cc metadata table
    message-attachment-list.js # Attachment chips with download links
    conversation-list-item.js # Thread row in master panel (avatar, subject, badge)
    message-thread.js        # Wraps message list for a thread
    message-date-separator.js # Date label between days in thread view
    mail-admin-tab.js        # Mailbox admin: IMAP account form, IMAP health, folder→entity assignments, chunk sync + wipe-resync, MailAdminClassificationRules (rule CRUD + reorder)
    mailbox-imap-status.js   # IMAP status presentation helpers (split from mail-admin-tab.js)
    mailbox-sync-controls.js # useMailboxSync (quick/chunk sync, wipe) + SyncConfirmModals (wipe only)
    management-page.js       # /verwaltung integrations orchestrator (delegates to sub-components)
    management-integration-card.js # IntegrationServiceCard shell (icon, title, badge, header actions)
    ksef-settings.js         # KSeF config (split from management-page.js)
    tracking-time-settings.js # TrackingTime config (split from management-page.js)
    youtube-settings.js      # YouTube OAuth + config (split from management-page.js)
    ai-provider-settings.js  # Anthropic/Ollama config (split from management-page.js)
    ai-profiles-page.js      # AI profiles: Ollama corpus runs, grounding, promote (Verwaltung → AI-Profile)
    admin-settings-page.js   # Mounts AdminSettingsPage: tabs buchhaltung/projekte/marketing/ai-anbindungen/ai-profiles/nachrichten; AI-Anbindungen + AI-Profile eigene Seiten
    marketing.js             # Marketing module (/marketing/start, /marketing/youtube)
    ai-chat-panel.js         # Claude chat UI (agentKey prop)
    master-detail-layout.js  # MasterDetailLayout — two-column master/detail generic wrapper
    scroll-panel.js          # ScrollPanel — scrollable flex child with optional header/footer slots
    list-search-field.js     # Debounced list search field
    table-pagination.js      # TablePagination — prev/next chevrons + page numbers between (DEFAULT_TABLE_PAGE_SIZE=200)
    searchable-select.js     # SearchableSelect + SearchableSelectStandalone
    icons.js                 # All shared SVG icons (IconDownload, IconEdit, IconTrash, IconAttachment, …)
    toggle.js                # Custom toggle switch
    dropdown.js              # Custom dropdown — trigger + floating menu
    skeleton.js              # SkeletonRows for table loading states
    toast-container.js       # Fixed bottom-right toast stack
    loader.js                # Spinning ring loader
    delete-modal.js          # Reusable confirm-delete / confirm-action modal (description prop for custom body)
    email-modal.js           # Email send modal
    login-form.js            # Login page form
  hooks/
    use-toast.js             # useToast() → { toasts, addToast }
    use-rest-action.js       # useRestAction() → { run, loading } — wraps apiFetch with loading state
    use-entity-crud.js       # useEntityCrud(postType, listQuery, entityLabel) — all-in-one hook for CPT list pages:
                             #   sidebar form state, delete modal, active-row toggle, toast, cache invalidation
    use-form-submit.js       # useFormSubmit({ postType, databaseId, entityLabel, buildPayload, onSuccess }) —
                             #   entity record loading, isLoaded gate, handleSubmit with create/update, toast, error formatting
    use-dls-query.js         # useDlsQuery(fetcher, deps) — lightweight fetch hook for custom REST (dls/v1):
                             #   { data, loading, error, refetch } — re-fetches when deps change
  services/
    entity-service.js        # EntityService — useEntityRecords wrapper for CPTs
    option-service.js        # JS option service
  helpers/
    currency.js              # formatToEuroCurrency, formatWithCurrency, getPriceInEur, formatFloatToString, formatStringToFloat
    dates.js                 # parseDateLocal, acfDateToTimestamp, acfDmYToIso, isoToAcfDmY, getMonthsBetweenDates, …
    data-utils.js            # sumPricesByMonth, getArrayDifference
    acf-normalize.js         # acfTrueFalseTo01, acfTrueFalseFromApi, normalizeClientAcfForRest, normalizeProductAcfForRest, …
    rest.js                  # formatRestSaveError
    contact.js               # normalizeEmailAddress, getPersonRecordsForClient, buildClientPortalWelcomeMailtoUrl, buildInvoiceMailtoHref, …
    mail-admin-shared.js     # mailPath(), mailboxSchema (Yup), clientLabel()
    mail-conversation-resolution.js # Thread helpers: normalizeMailAddress, isInternalOwnerEmailNormalized,
                             #   resolveThreadContactPerson, threadFallbackContactLabel,
                             #   conversationListAvatarInfo, threadSubjectPreview, personDisplayName
    email-classification-labels.js # English token → German label maps for categories, actions, urgency, condition types, source; emailClassificationDisplayDe(token, kind); EMAIL_CATEGORY_COLOR
    html-mail-plain-text.js  # Plain-text extraction from email HTML
  queries.js                 # Centralised list-query constants: TRANSACTION_LIST_QUERY, INVOICE_LIST_QUERY,
                             #   CLIENT_LIST_QUERY, PERSON_LIST_QUERY, FILE_LIST_QUERY, PRODUCT_LIST_QUERY,
                             #   ACCOUNTING_CONFIG_LIST_QUERY, CREDENTIALS_LIST_QUERY
  helpers.js                 # Barrel re-export from helpers/*.js — parseDateLocal, formatWithCurrency, getPriceInEur, …
build/                       # Compiled webpack output (do not edit — run `npm run build`)
assets/scss/                 # SCSS source (ScssPhp, compiled on theme load when SCSS is newer)
  variables.scss             # All colour/font tokens — import first in every partial
  styles.scss                # Global styles, buttons, modals, layout
  card.scss / grid.scss / form.scss
  components/                # Per-feature SCSS partials
    messages.scss            # Core mail styles (imports inbox, mail-admin-panel, chunk-sync)
    mail-admin-panel.scss    # Mailbox admin panel styles (split from messages.scss)
    inbox.scss               # Inbox/thread list styles (split from messages.scss)
    chunk-sync.scss          # Chunk sync progress styles (split from messages.scss)
    conversation-layout.scss # MasterDetailLayout + ScrollPanel + thread + bubble styles
    filter-bar.scss          # Filter bar (inbox top bar)
    management-page.scss     # Integration card + AdminSettingsPage + .dls-email-analysis-type-row + writing-style / classification UI blocks
    entity-panel.scss        # EntityListPanel (.entity-panel, __head, __title, __loader, __empty)
    list-search-field.scss   # ListSearchField (.field-container.dls-list-search-field)
    table-pagination.scss    # TablePagination (.dls-table-pagination)
    formik-date-picker.scss  # FormikDatePicker (.formik-datepicker)
    ai-chat-panel.scss       # AI chat panel styles
    client-card.scss         # Client card expanded view
    client-form.scss         # Client form sidebar overrides
    file-card.scss           # File card styles
    media-modal.scss         # Media/image picker modal
    person-card.scss         # Person card styles
    portal-credentials-modal.scss # Portal credentials modal
    project.scss             # Project board layout
    skeleton.scss            # SkeletonRows loading states
    toast.scss               # Toast notification styles
    toggle.scss              # Toggle switch styles
    document-list.scss / projects.scss / accounting.scss / …
```

## Virtual app routes (no WP page)
- File: `inc/app-routes.php` — `dls_get_app_routes()` + `add_rewrite_rule` for `/projects`, `/marketing`, `/nachrichten`, `/verwaltung`, `/werkzeuge`.
- `/verwaltung` has sub-routes: `buchhaltung`, `projekte`, `marketing`, `ai-anbindungen`, `ai-profiles`, `nachrichten` (tab selection via `data-subroute`). After changing sub-route slugs, flush permalinks (Settings → Permalinks → Save).
- `/marketing` has sub-routes: `start` (default; `/marketing/` redirects), `youtube` — PHP `.sub-header` tabs + `.site-meta` h1 + `data-subroute` on `.dls-marketing` (same shell as Buchhaltung + Verwaltung tabs).
- `/werkzeuge` has sub-routes: `email-migration` (default; `/werkzeuge/` opens it) — PHP `.sub-header` + `.site-meta` h1 + `tools-page.js` on `.dls-werkzeuge` (`data-subroute`). Nav link in `header.php`.
- **E-Mail-Migration** (`email-migration-tab.js`): **`POST /dls/v1/imap-sync/start`**, **`GET /dls/v1/imap-sync/jobs`**, **`GET /dls/v1/imap-sync/config`** (health), **`DELETE /dls/v1/imap-sync/jobs/{id}`** (cancel). Proxy **`inc/routes/imap-sync-proxy.php`**, option **`dls_imap_sync_base_url`**. Stella uses temp passfiles for `imapsync`. Separate from Nachrichten DB import.
- `AdminSettingsPage` in `admin-settings-page.js`: switches between `MailAdminMailboxes` (nachrichten), `AiConnectorsPage` / `AiProfilesPage` (AI tabs), and `ManagementPage` (buchhaltung/projekte/marketing).
- Add new screens: add to `dls_get_app_routes()`, add `add_rewrite_rule`, add to `admin-settings-page.js` tab list if it's a Verwaltung tab, mount React component. Flush permalinks after adding.
- **Do not** create a WP page with the same slug.

## Nachrichten (IMAP)

**Schema version:** `DLS_MAIL_DB_VERSION = 31` (option `dls_mail_db_version`). Legacy tables (`dls_mailbox`, `dls_email`, spam blocklist/whitelist, embed queue) were dropped in the v3 migration.

- **Tables** (`inc/install-mail-tables.php`):

| Table | Purpose |
|---|---|
| `dls_mail_account` | IMAP accounts — credentials (password AES-encrypted), active flag |
| `dls_mail_folder` | Per-account folder rows: `uid_validity`, `category`, `last_synced_at` |
| `dls_mail_message` | One row per `(account_id, folder_id, imap_uid)`; `direction` inbound/outbound; `thread_id`; classification columns `email_category`, `email_action`, `email_urgency`, `classification_source` (`rule`/`ai`); `has_attachment`, `is_seen`, `is_flagged`, `subject_normalized` |
| `dls_mail_folder_link` | Polymorphic folder → entity mapping |
| `dls_mail_message_link` | Polymorphic message → entity; `source`: `folder` / `manual` / `address` |
| `dls_mail_attachment` | Attachment metadata; files under `wp-content/private/dls-mail-attachments/{account_id}/{message_id}/` |
| `dls_mail_classification_rule` | Ordered Layer-1 classification rules |
| `dls_mail_sync_run` | Chunk sync audit log — `status` running → done/error/cancelled; `chunk_size`, `uid_cap`, `total_queued`, `processed_count`, `deleted_count`, `restored_links`, `errors_json`, timestamps |

- **Services** (`inc/services/`): `MailDbService` (DB façade), `MailSyncV2` (sync engine), `MailSyncRunDbService` (chunk audit log), `MailAttachmentService` (private attachment storage), `MailClassificationService` (rule + Ollama classification), `MailCrypto` (password AES), `DlsImapFactory` (Webklex PHPIMAP).
- **WP-Cron:** `inc/mail-sync-cron.php` — hook `dls_mail_imap_sync_cron` every **5 minutes** (`dls_every_five_minutes`); calls `MailSyncV2::sync_account($id, $limit)` for each active account (default limit 100, filterable via `dls_mail_cron_sync_limit`, min 10).
- **Quick sync vs chunk sync:**
  - **Quick** (`sync_account`): per-folder UID diff, handles UIDVALIDITY change (purge + reimport), flag sync up to 500 UIDs, imports new UIDs up to `$limit`. Used by cron and `POST /mailboxes/{id}/sync`.
  - **Chunk** (full wipe-resync): `start_chunk_job` wipes all mailbox messages, saves manual links to transient, builds per-folder UID queues, logs run in `MailSyncRunDbService`. `process_chunk_step` imports one folder's batch. On completion: `restore_manual_links_after_resync`. Job in transient `dls_mail_sync_job_{id}` (3 h TTL). Default `uid_cap = 0` (unlimited).
- **Direction:** `outbound` when IMAP path matches Sent folder (`sent` / `gesendete`) or From matches `DLS_OWNER_EMAILS` / `DLS_OWNER_DOMAINS`; else `inbound`. **Classification** (`MailClassificationService`) is off by default (`DLS_MAIL_CLASSIFICATION_ENABLED`); when enabled, only **main INBOX** + **inbound** messages are classified (not subfolders, not outbound).
- **Classification** (`MailClassificationService`): Layer-1 ordered rules in `dls_mail_classification_rule` (first match; `classification_source = 'rule'`); Layer-2 Ollama synchronous call if no rule matches and `AiRuntimeCredentials::ollama_base_url()` is set (`classification_source = 'ai'`). Runs **synchronously** inside `import_uid_batch`.
- **Client assignment** — links in `dls_mail_message_link` with `source`:
  1. **`folder`** — folder→entity from `dls_mail_folder_link`; highest priority.
  2. **`address`** — From/To matching against `MailDbService::build_client_email_map` (WP CPTs: client emails, people, invoice emails).
  3. **`manual`** — user-set via `PUT /emails/{id}`; survives wipe-resync via transient restore.
  - `recompute_client_assignments` deletes `folder` + `address` links and re-derives them; manual links are preserved.
- **REST** (all `dls/v1`): `mailboxes.php` (mailbox CRUD, folder-entity assignments, IMAP health/test/list-folders, `folder-clients-overview`), `mail-emails.php` (email list/get/update), `mail-sync.php` (quick sync, chunk sync start/step/cancel/status, wipe, recompute-metadata, clear-client-links), `mail-classification.php` (classification rules CRUD + reorder, POST classify).
- **UI:** `messages-page.js` (dark inbox, thread grouping by `thread_id`, tabs Kunden/WordPress/Sonstiges, filters, pagination), `mail-admin-tab.js` (mailbox CRUD + classification rules), `mailbox-sync-controls.js` (quick/chunk/wipe sync + resume), `email-classification-labels.js` (token → German label maps), `mail-conversation-resolution.js` (thread helpers, avatar, reply builder). SCSS: `messages-page.scss`, `mail-admin-panel.scss`, `inbox.scss`, `chunk-sync.scss`, `filter-bar.scss`, `conversation-layout.scss` (imported by `messages.scss`).

## E-Mail-AI-Analysen (Ollama, Verwaltung → AI-Profile)

**UI:** Tab **AI-Profile** (`/verwaltung/ai-profiles/`) — `ai-profiles-page.js`: Karten-Footer **Bearbeiten** + **Analyse starten** (gespeicherte Filter/Modell); Sidebar **Profil bearbeiten** mit Prompts, Filtern, Speichern, **Vergangene Analysen** (EntityListTable), **Übernehmen** (promote) — Laufstart nur von der Karte. Es darf **global nur ein** Corpus-Lauf gleichzeitig laufen — zweiter Start liefert **409** (`analysis_busy`), sofern nicht derselbe Profil-Lauf bereits läuft (dann **200** mit Status). **Anbindungen** (Ollama-URL) liegen auf **AI-Anbindungen** (`dls_ai_connector` + `AiRuntimeCredentials`). Die Layer-Metadaten (`dls_ai_analysis_config`, `dls_ai_action`) bleiben für andere Features; Schreibstil-/Klassifizierungs-Grounding wird primär in **`dls_ai_profile`** geführt und bei PATCH auf `reply_style_grounding` / `inbox_classification_grounding` aus den Aktionen mit synchronisiert (`AiActionDbService::sync_legacy_options_from_row` spiegelt auch in die Profil-Zeile).

### Tabellen & Install

- **`dls_ai_profile`** / **`dls_ai_profile_run`:** `inc/install-ai-profile-tables.php` (Option `dls_ai_profile_db_version`). Ein Profil je `(name, source_type)` — Seeds liefern weiter `system_prompt` + `user_prompt`; **Verwaltungs-UI** bearbeitet nur **`user_prompt`** (Analyse-Prompt) und leert `system_prompt` beim Speichern. Ollama-Aufruf (`ai-profile-run-execute.php`): System-Rolle nur wenn `system_prompt` nicht leer (Legacy), sonst nur User-Nachricht aus `user_prompt` (Platzhalter `{{corpus}}`, `{{count}}`, `{{model}}`). Filter-JSON u. a. `direction`, `spam_status`, `sample_limit`, `min_body_chars`, `max_body_chars`; Klassifizierung: `require_substring` für Output-Check.
- **Worker:** `inc/scripts/run-ai-profile.php --run-id=N` — Logik in `inc/ai-profile-run-execute.php` (Corpus → Platzhalter ersetzen → Ollama `/api/chat`, Timeout 14400s). Corpus-Builder: `AiEmailCorpusBuilder` + `AiCorpusBuilderRegistry` (`task` / `slack` → noch nicht implementiert).
- **REST (neu):** `inc/routes/ai-profiles.php` — `GET/PATCH /dls/v1/ai/profiles`, `GET /dls/v1/ai/profiles/{id}`, `POST /dls/v1/ai/profiles/{id}/run`, `POST /dls/v1/ai/profiles/runs/{id}/promote`, `POST /dls/v1/ai/profiles/worker/stop`. Worker-State: Option `dls_ai_profile_worker_state`; Cron-Fallback: `dls_ai_profile_cron_worker`.

### Legacy-Historie & kompatible Endpunkte

- **`dls_writing_style_run`:** bleibt bestehen; erfolgreiche/fehlgeschlagene Läufe des neuen Workers rufen weiter `WritingStyleHistoryService` auf, sodass `GET /dls/v1/ai/writing-style-history` gefüllt bleibt. Einmalige Migration alter Zeilen nach `dls_ai_profile_run` beim Install, wenn die neue Run-Tabelle leer ist.
- **Schreibstil:** `inc/routes/ai-writing-style.php` — `GET/POST`-Endpunkte lesen bzw. starten über **`dls_ai_profile`** / `dls_ai_profile_begin_run_for_profile_id`; `dls_get_active_writing_style_grounding()` bevorzugt **`dls_ai_profile.grounding`**, dann Legacy.
- **Klassifizierung:** `inc/routes/ai-email-classification-suggestions.php` — analog; `dls_get_active_email_classification_grounding()` bevorzugt Profil-Grounding; `GET email-classification-profile` mappt aus Runs + Profil.

### Alte CLI-Skripte

- `learn-writing-style.php` / `learn-email-classification-suggestions.php` werden von den REST-Startern nicht mehr verwendet; Referenz für Prompt-Texte. Optional manuell aufrufbar, solange die alten Status-Optionen nicht gesetzt sind.

## Project Management (PM)

- **Tables** (`inc/install-pm-tables.php`, option `dls_pm_db_version` v2): `dls_pm_project` (name, status, client_id, tt_project_id), `dls_pm_task_list` (project_id, sort_order, tt_task_list_id), `dls_pm_task` (title, status, due_date, sort_order, tt_task_id), `dls_pm_task_assignee` (task_id, user_id), `dls_pm_comment` (task_id, user_id, body, tt_comment_id), `dls_pm_time_entry` (task_id, project_id, user_id, minutes, started_at, tt_time_entry_id).
- **Service:** `DLS\Services\PmDbService` (`inc/services/pm-db-service.php`) — full CRUD for projects, task lists, tasks, assignees, comments, time entries. `get_project_id_by_tt_project_id()` for TrackingTime import upsert.
- **REST:** `inc/routes/pm-projects.php` — namespace `dls/v1`, prefix `/pm`; routes for projects, task lists, tasks, assignees, comments, time entries. Auth: `dls_pm_permission()` (logged-in).
- **UI:** `src/components/project-management.js` — mounts on `.dls-project-management`; full CRUD UI with `SidebarForm`, `SearchableSelect` (client), `DeleteModal`, task lists + tasks inline.
- **Virtual route:** `/projects` → `src/index.js` → `project-management.js`. Also accessible as sub-route `projekte` on `/verwaltung/projekte`.
- **TrackingTime link:** `tt_project_id`, `tt_task_id`, etc. stored for idempotent import from TrackingTime Pro API (`inc/routes/tracking-time.php`).

## KSeF Integration

### Overview
Service: `inc/services/ksef-service.php` — class `DLS\Services\KSeFService` (namespace `DLS\Services`).
Supports both **importing** (receiving) and **sending** (outgoing) invoices via KSeF API 2.0.

### Environment
- **Test mode** (default): `https://api-test.ksef.mf.gov.pl/api/v2` — controlled by `ksef_test_mode` option.
- **Production**: `https://api.ksef.mf.gov.pl/api/v2` — switch off test mode in Verwaltung → KSeF settings.

### Auth
6-step async JWT flow (challenge → RSA-OAEP-SHA256 encrypt → POST `/auth/ksef-token` → poll status → redeem → `accessToken`). Same JWT used for both import and send operations.

### Importing (receiving invoices)
- Import window: always fixed 3-month range (first day of 2 months ago → last day of current month). Frontend date params are ignored.
- Upsert behaviour: **on create** — write all fields including `note`, `type`, `post_title`; **on update** — only refresh financial/date fields (`invoice_date`, `price_net`, `vat_percent`, `currency`) and KSeF identifiers. Never overwrite user-edited `note` or `post_title` on re-import.
- Seller name rules (transaction `note` on **create** only): if seller name starts with `"Good Code"` → note `"Freelancer (Pawel)"`; if it starts with `"KANCELARIA PODATKOWA ANDRZEJ NOWAK SPÓŁKA Z OGRANICZONĄ ODPOWIEDZIALNOŚCIĄ"` → note `"Accounting KPAN"` + invoice number (trimmed).
- `ksef_reference` and `ksef_seller_nip` are **ACF fields** (field keys `field_6829ksef000r1` / `field_6829ksef000n2`) — use `update_field($field_key, $value, $post_id)`, not `update_post_meta`.
- PLN conversion rate is auto-populated via `NbpExchangeRateService::get_rate_for_input_date($issue_date)` and stored in `pln_conversion_rate_invoice_date`.

### Sending (outgoing invoices)

Mandatory from **2026-02-01** (large enterprises) / **2026-04-01** (all). FA(3) schema only.

- **XML Builder**: `inc/services/ksef-xml-builder.php` — class `DLS\Services\KSeFXmlBuilder`. Generates FA(3) compliant XML from `invoice` CPT data (schema: `http://crd.gov.pl/wzor/2025/06/25/13775/`). When the linked **Einnahme** transaction has ACF `income_category` = `Werbeinnahmen` (legacy stored value `Werbung` still triggers GTU), each `FaWiersz` includes optional `GTU` = `GTU_12` (JPK_V7 — advertising / marketing-type services).
- **REST route**: `POST /dls/v1/ksef-send/{invoice_id}` — builds XML, validates, sends via interactive session, updates ACF tracking fields.
- **Status check**: `GET /dls/v1/ksef-send-status/{invoice_id}` — polls KSeF session-based status, updates `ksef_invoice_number` on success.
- **Duplicate prevention**: refuses to send if `ksef_send_status === 'sent'` (HTTP 409).
- **Validation**: structural XML validation via `KSeFXmlBuilder::validate_xml()` before sending (checks required elements, NIP format, date format, P_12 rate values).

#### Interactive session flow (KSeF API 2.0)

Sending uses **encrypted interactive sessions** — no direct `PUT /invoices`:

1. **Authenticate** — JWT via existing ksef-token flow (same `accessToken` for all session operations).
2. **Fetch encryption cert** — `GET /security/public-key-certificates` with `usage: SymmetricKeyEncryption` (different from `KsefTokenEncryption` used in auth).
3. **Generate AES key + IV** — `random_bytes(32)` for AES-256 key, `random_bytes(16)` for CBC IV.
4. **Encrypt AES key** — RSA-OAEP-SHA256 with the SymmetricKeyEncryption public key.
5. **Open session** — `POST /sessions/online` with `formCode` (`FA (3)`, schema `1-0E`), `encryption` (encrypted key + IV in base64).
6. **Encrypt invoice** — AES-256-CBC with PKCS#7 padding (`openssl_encrypt` with `OPENSSL_RAW_DATA`).
7. **Submit** — `POST /sessions/online/{sessionRef}/invoices` with JSON payload:
   - `invoiceHash` — SHA-256 of plain XML (base64)
   - `invoiceSize` — byte length of plain XML
   - `encryptedInvoiceHash` — SHA-256 of ciphertext (base64)
   - `encryptedInvoiceSize` — byte length of ciphertext
   - `encryptedInvoiceContent` — ciphertext (base64)
   - `offlineMode` — `false`
8. **Close session** — `POST /sessions/online/{sessionRef}/close` (204 No Content on success).
9. **Poll status** — `GET /sessions/{sessionRef}/invoices/{invoiceRef}` — up to 5 attempts (2 s apart) inline; frontend polls via REST if still pending.

#### Reference storage

`ksef_reference_number` stores a **JSON string** with both identifiers needed for status polling:
```json
{"session": "<sessionRef>", "invoice": "<invoiceRef>"}
```
The status endpoint parses this and passes both refs to `KSeFService::check_send_status()`.

#### VAT rate mapping
- `vat_pct > 0` → rate string (e.g. `"23"`)
- `vat_free === "Drittland"` → `"np II"` (third country, outside EU)
- `vat_free === "Reverse Charge"` → `"np I"` (EU intra-community)
- Non-PL EU buyer (fallback) → `"np I"`
- Non-PL non-EU buyer (fallback) → `"np II"`

#### Buyer identification (Podmiot2)
- PL buyer: `<NIP>` (10-digit, extracted from `vat_number`)
- EU buyer: `<KodUE>` + `<NrVatUE>` (e.g. `DE` + `DE123456789`)
- Non-EU buyer: `<KodKraju>` + `<NrID>` (country ISO code + tax ID)

- **Country resolution**: `KSeFXmlBuilder::resolve_country_code()` maps free-text country names (German, English, Polish) to ISO alpha-2 codes.
- **ACF tracking fields** on `invoice` CPT:
  - `ksef_invoice_number` (`field_6961inv_ksef_number`) — KSeF number assigned after successful send
  - `ksef_send_status` (`field_6961inv_ksef_status`) — `""` / `"pending"` / `"sent"` / `"error"`
  - `ksef_sent_at` (`field_6961inv_ksef_sent_at`) — ISO 8601 timestamp of last send attempt
  - `ksef_error_message` (`field_6961inv_ksef_error`) — last error from KSeF API
  - `ksef_reference_number` (`field_6961inv_ksef_ref`) — JSON `{"session":"…","invoice":"…"}` for session-based status polling
- **Schema files**: `inc/ksef/schemat.xsd` (FA(3) XSD), `inc/ksef/styl.xsl` (XSLT for HTML preview), `inc/ksef/wyroznik.xml` (schema reference).

#### FA(3) mandatory fields in generated XML

Beyond the basic structure, the following elements are mandatory in FA(3) schema `1-0E` and must always be present:

- **`Podmiot2` (buyer):** `<JST>2</JST>` + `<GV>2</GV>` after `<Adres>` — required by schema even when buyer is not a local government unit or value-added goods dealer.
- **`Adnotacje`:** `<NoweSrodkiTransportu><P_22N>1</P_22N></NoweSrodkiTransportu>`, `<P_23>2</P_23>`, and `<PMarzy><P_PMarzyN>1</P_PMarzyN></PMarzy>` — always required markers for new means of transport, tourist services, and margin procedures respectively (value `1`/`2` = "does not apply").

#### Re-sending between environments

- A custom post meta key `_ksef_sent_env` (`production` or `test`) tracks which environment an invoice was last sent to.
- The send endpoint (`POST /dls/v1/ksef-send/{id}`) allows re-sending if the target environment **differs** from the stored `_ksef_sent_env` — it clears all previous KSeF references (`ksef_send_status`, `ksef_reference_number`, etc.) before the new send.
- Re-sending to the **same** environment when `ksef_send_status === 'sent'` is blocked with HTTP 409.
- The KSeF send button in `invoices.js` is **always visible** — the backend controls the allow/block logic.

### Settings (WP options)
- `ksef_enabled` (bool) — master toggle
- `ksef_nip` (string) — 10-digit NIP for authentication
- `ksef_token` (string) — KSeF portal authorization token for **production** (`https://ap.ksef.mf.gov.pl/`)
- `ksef_test_token` (string) — separate token for **test environment** (`https://ap-test.ksef.mf.gov.pl/`); generated independently at the KSeF test portal
- `ksef_test_mode` (bool, default `true`) — use test environment; `KSeFService` selects the matching token automatically
- `ksef_seller_name` (string) — company name for FA(3) Podmiot1
- `ksef_seller_address` (string) — street address for FA(3) Podmiot1
- `ksef_seller_city` (string) — "PLZ Ort" for FA(3) Podmiot1

### Frontend
- `ksef-settings.js` — KSeF config card in Verwaltung: NIP, separate production/test tokens, test mode toggle (shown last), seller identity fields, save, test connection. Composed inside `.dls-buchhaltung-body__ksef` in `management-page.js` for side-by-side layout with bank accounts.
- `invoices.js` — KSeF column with status badges (Gesendet/Wird verarbeitet/Fehler), send button (IconLandmark) **always visible** (backend controls permission), detail view shows KSeF number + error message.

## Bank Accounts (`dls_bank_account`)

### Overview

Bank accounts for invoice and PDF generation are stored in a **custom database table** (`dls_bank_account`) managed via `inc/install-bank-account-tables.php` (option `dls_bank_account_db_version`, current v3). A CRUD REST API and a React UI in Verwaltung → Buchhaltung allow adding, editing, and deleting entries.

### Database table

```sql
CREATE TABLE dls_bank_account (
  id            bigint(20) NOT NULL AUTO_INCREMENT PRIMARY KEY,
  label         varchar(255) NOT NULL DEFAULT '',
  iban          varchar(64)  NOT NULL DEFAULT '',
  bic           varchar(32)  NOT NULL DEFAULT '',
  bank_name     varchar(255) NOT NULL DEFAULT '',
  account_holder varchar(255) NOT NULL DEFAULT '',
  condition_country varchar(16) NOT NULL DEFAULT '',   -- ISO alpha-2 or '' (any)
  condition_vat int(11)      NOT NULL DEFAULT -1,      -- VAT % or -1 (any)
  currency      varchar(8)   NOT NULL DEFAULT '',      -- EUR, PLN, USD, …
  is_primary    tinyint(1)   NOT NULL DEFAULT 0,       -- one primary per currency
  -- US-specific fields
  ach_routing   varchar(32)  NOT NULL DEFAULT '',
  wire_routing  varchar(32)  NOT NULL DEFAULT '',
  account_number varchar(64) NOT NULL DEFAULT '',
  account_type  varchar(16)  NOT NULL DEFAULT ''       -- 'checking' | 'savings' | ''
);
```

### Service — `BankAccountDbService`

File: `inc/services/bank-account-db-service.php` (class `DLS\Services\BankAccountDbService`).

- `list_accounts()` — all rows ordered by `id ASC`
- `get_account(int $id)` — single row or null
- `create_account(array $data)` / `update_account(int $id, array $data)` / `delete_account(int $id)` — CRUD
- `find_account(string $currency, string $country, int $vat_pct)` — intelligent lookup:
  1. Filter by exact `currency` match (returns `null` if no account has that currency)
  2. Among matching accounts, prefer `condition_country` match over `''` (any)
  3. Among those, prefer `condition_vat` match over `-1` (any)
  4. Returns the best match or the first currency-match as fallback
- `set_primary(int $id, string $currency)` — unsets `is_primary` for all accounts with the same currency, then sets it for `$id`
- `get_primary_for_currency(string $currency)` — returns the account with `is_primary = 1` for that currency, or null

### REST API

File: `inc/routes/bank-accounts.php` — registered under `dls/v1`. Auth: `dls_require_login`.

| Method | Path | Action |
|--------|------|--------|
| `GET` | `/dls/v1/bank-accounts` | List all accounts |
| `POST` | `/dls/v1/bank-accounts` | Create (calls `set_primary` if `is_primary === true`) |
| `PUT` | `/dls/v1/bank-accounts/{id}` | Update (calls `set_primary` if `is_primary === true`) |
| `DELETE` | `/dls/v1/bank-accounts/{id}` | Delete |
### Bank account selection priority

Both PDF and KSeF XML generation use the same three-tier priority chain:

1. **Invoice pin** — ACF field `bank_account_id` on the `invoice` CPT (`field_6961inv_bank_acct`, select of `dls_bank_account` row ids; `0` = no pin). Read in PHP via `get_field( 'bank_account_id', $invoice_id )`.
2. **Transaction pin** — `bank_account_id` registered post meta on the linked `transaction` CPT (`register_post_meta` in `functions.php`, `show_in_rest: true`).
3. **Auto-select** — `BankAccountDbService::find_account($currency, $buyer_country, $vat_pct)`

Invoice bank choices are built in `inc/post-types/invoice.php` (`acf/load_field/name=bank_account_id` → `BankAccountDbService::list_accounts()`). The React invoice form saves `bank_account_id` inside the normal `acf` payload on `saveEntityRecord` and invalidates `getEntityRecord` after save.

### Frontend

- `bank-account-settings.js` — `EntityListPanel`-based CRUD UI rendered in `.dls-buchhaltung-body__accounts` (side-by-side with KSeF settings). List rows show label + currency badge + optional "Primär" badge, IBAN/account number below. Sidebar form with all fields; US-specific fields shown only when `condition_country === 'USA'`. `is_primary` toggle auto-unsets the previous primary for that currency on save.
- `invoice-form.js` — bank account native `<select>` (`InvoiceBankAccountSelect`), backed by `acf.bank_account_id`. If the invoice has **no** stored pin (&gt; 0 in ACF), `InvoiceBankAccountLanguageSuggest` picks a default from **Sprache** (DE→EUR-first, PL→PLN-first, EN→EUR/USD) using `suggestBankAccountId()` in `src/helpers/invoice-bank-account.js`. Changing **Sprache** refreshes the suggestion unless the user already chose a non-empty account for the same language. A stored ACF value is never overwritten by the effect. `invoices.js` passes `key={mountKey}` so the form resets when opening the sidebar.
- `management-page.js` — Buchhaltung tab wraps both `KsefSettings` and `BankAccountSettings` in `div.dls-buchhaltung-body` (flexbox, side-by-side, wraps on narrow screens).
- `management-page.scss` — `.dls-buchhaltung-body { display: flex; flex-wrap: wrap; gap: …; }` with `__ksef` (fixed width) and `__accounts` (`flex: 1 1 360px`) children.

### Seed data

`dls_bank_account_seed_defaults()` (called on first install / when table is empty) seeds the initial bank accounts from the previously hardcoded PDF generation values (AT/EUR, PL/PLN, USA/USD).

## Referral / Commission System
- ACF fields on `client` post type: `referred_client` (bool), `referral_referrer_client_id` (int), `referral_commission_percent` (float, default 10), `referral_commission_payout_date` (date), `referral_commission_reports` (repeater: `generated_at` datetime, `pdf_url` url) — auto-appended when a PDF is generated
- `inc/post-types/client.php` — `acf/save_post` hook clears referral fields when `referred_client` is unchecked (does not clear `referral_commission_reports`)
- `inc/commission-reports-storage.php` — `dls_commission_reports_storage()`, `dls_commission_reports_ensure_dir()` — PDFs under `wp-content/uploads/commission-reports/` (folder created if missing) with unique SHA-256 filenames (never overwrite)
- Commission report PDF: `CommissionReportService::create_referral_commission_report_pdf_persistent(int $referrer_id)` — persists file + used by `GET /dls/v1/referral-commission-report/{id}` (also `add_row` on the referrer client). Legacy: `create_referral_commission_report_pdf_to_temp` if needed for one-off temp output
- `DELETE /dls/v1/referral-commission-report/{id}/{row}` — `id` = referrer client ID, `row` = **1-based** ACF repeater index for `referral_commission_reports`; deletes the row, unlinks the PDF under uploads when URL is safe (`dls_commission_report_pdf_path_from_public_url`). Auth: logged-in users
- Commission report HTML: `CommissionReportService::build_referral_commission_report_html(int $referrer_id)` — used by the PDF generator
- `client-form.js` — on update, merges `client.acf.referral_commission_reports` into the save payload so the React form does not wipe server-managed rows
- Only invoices with `invoice_date < referral_commission_payout_date` (cutoff) are included
- Rows sorted oldest → newest by actual invoice date (not WP post date)

## PDF Generation (Spipu Html2Pdf / TCPDF) — Critical Patterns & Gotchas

### Shared infrastructure in `FileService`
- `get_pdf_base_css(string $extra_css)` — shared `<style>` block for all PDFs; pass document-specific CSS via `$extra_css`
- `get_pdf_header_html(...)` — shared dark-background header (customer info + profile + Dominik info + invoice-meta row); used by invoices AND commission report
- Invoice PDFs use `[0, 0, 0, 0]` constructor margins (edge-to-edge layout)
- **Invoice `InvoicePdfService`**: If `service_start_date` and `service_end_date` fall on the same calendar day, the PDF shows **Leistungsdatum** / **Service date** / **Data sprzedaży** (PL) with a single formatted date instead of **Leistungszeitraum** with a range. When that day matches the invoice date, the value line still uses the existing “same as invoice date” wording (DE/EN/PL). Position override text, product title, and product `description` are passed through `wp_kses` with a small inline tag allowlist (`br`, `strong`, `b`, `em`, `i`, `u`, `p`, `span`) so Html2Pdf renders formatting instead of escaping tags as plain text. KSeF `FaWiersz` / `P_7` uses plain text (`wp_strip_all_tags` on the override description) because FA XML must not carry HTML markup.
- Commission report uses `[13, 13, 13, 13]` constructor margins + `<page backbottom="10mm">` + `<page_footer>[[page_cu]]/[[page_nb]]</page_footer>` for page numbering

### Page margins + edge-to-edge header
- The commission report header uses `position: relative; top: -13mm; left: -13mm` in `$report_extra_css` to escape the 13mm page margins, matching the proposal template
- **CRITICAL**: This CSS MUST be defined as a single complete `div.header { }` rule in `$report_extra_css` — do NOT split properties across two rules. Html2Pdf's CSS parser does not reliably cascade two rules for the same selector; the second rule may completely replace the first rather than merging, causing layout failures
- **CRITICAL**: Do NOT wrap the first client section's content in a `page-break-inside: avoid` div. The `position: relative; top: -13mm` on the header affects Html2Pdf's flow cursor, leaving less apparent space on page 1. A `page-break-inside: avoid` container on the first section will cause Html2Pdf to move ALL content to page 2. Subsequent client sections CAN use `page-break-inside: avoid`
- Do NOT wrap body content in a `div.rpt-body` container — content should flow directly after the header with no wrapper div

### Html2Pdf layout rules
- Avoid `<div>` nesting for layout — prefer flat `<table>` elements for each row; deep nesting causes page-break and overlap bugs
- Each invoice row should be its own `<table>` element (not rows in one large table) to prevent page-break issues
- Use `<table width="500px">` (fixed) for content width; all tables share `table { width: 500px; }` from `get_pdf_base_css`
- Column widths must account for cell padding: `(content_width + left_padding + right_padding)` summed across all columns must equal the table width
- `p { margin: 0; padding: 0; }` scoped carefully — a global reset interferes with header `<p>` elements
- Page-break grouping: wrap heading + th-row + first invoice row in `<div style="page-break-inside: avoid">` (skipped for first client); wrap last invoice row + subtotal row similarly
- Use `sys_get_temp_dir()` and `uniqid()` for temp file paths — do NOT use `wp_tempnam()` or `wp_generate_password()` in namespaced REST callback context (causes fatal errors)

## Adding New Routes
1. Create `inc/routes/your-route.php`
2. Use `dls/v1` namespace for browser-facing routes
3. Add `require` in `functions.php` alongside other route requires (routes still use manual require; services use Composer classmap autoloading)

---

## Checklist: Adding a new ACF field to a CPT

Use this checklist every time a field is added or renamed. Missing any step is the most common source of silent data loss or stale-state bugs.

**Backend (PHP)**
- [ ] Edit `inc/acf/settings/group_*.json` — add the field definition
- [ ] Sync the field group in WP Admin → Custom Fields → Sync (or save the group)
- [ ] If the field has business logic on save (e.g. clearing dependent fields), add an `acf/save_post` hook in the relevant `inc/post-types/*.php`

**JS — form wiring**
- [ ] Add to `initialValues` — **all three branches**: `databaseId === 0` (create), `!entity` (loading), and the populated branch
- [ ] Add to Yup `validationSchema` if user input must be validated — use `.when('otherField', …)` for conditional rules
- [ ] Add to `handleSubmit` ACF payload — apply any required type coercion (e.g. `isoToAcfDmY`, `parseInt`, `String`, `isTpl ? 0 : value`)
- [ ] Add `<ErrorMessage name="fieldName" component="div" className="validation-error" />` next to the field in JSX

**Cache invalidation**
- [ ] If the field is used as a filter or computed value in a list view, make sure `invalidateResolution` is called with the matching `*_LIST_QUERY` from `src/queries.js` after save

**Documentation**
- [ ] Add the field to **`cursor-rules/acf-field-map.md`** (and `.cursor/rules/acf-field-map.mdc` in the theme) under the correct CPT section (ACF type + return value / notes)

---

## Error Feedback — Single Pattern

All user-facing errors and success messages must use **`addToast`** from `useToast`. Never use `window.alert` or rely on `console.error` alone.

```js
import { useToast } from "../hooks/use-toast";
import ToastContainer from "./toast-container";

// Inside the component:
const { toasts, addToast } = useToast();

// On save success:
addToast(databaseId > 0 ? "Entity aktualisiert" : "Entity erstellt");
// On save error:
addToast(formatRestSaveError(err), "error");
```

Mount `<ToastContainer toasts={toasts} />` once at the top of every page-level component **and** every standalone form component (e.g. `FileForm`, `ProductForm`) that has its own `handleSubmit`. Forms embedded inside a parent that already has a `ToastContainer` may receive `addToast` via props instead of creating their own.

---

## Formik Validation Error Display — Single Pattern

All Formik field validation errors must use:

```jsx
<ErrorMessage name="fieldName" component="div" className="validation-error" />
```

Never use `{errors.field && <p>…</p>}` — `ErrorMessage` is `touched`-aware and only shows errors after the user has interacted with the field. Import from `formik`: `import { ErrorMessage } from "formik"`.

---

## Cache Invalidation Query Constants

All `invalidateResolution("getEntityRecords", …)` calls must use the named constants from **`src/queries.js`**:

```js
import { TRANSACTION_LIST_QUERY, INVOICE_LIST_QUERY, FILE_LIST_QUERY, CLIENT_LIST_QUERY,
         PERSON_LIST_QUERY, PRODUCT_LIST_QUERY, ACCOUNTING_CONFIG_LIST_QUERY, CREDENTIALS_LIST_QUERY } from "../queries";

invalidateResolution("getEntityRecords", ["postType", "transaction", TRANSACTION_LIST_QUERY]);
```

**Never inline** `{ per_page: -1, order: "asc", orderby: "title", acf_format: "standard" }` — a single copy-paste difference in key order or value will silently fail to invalidate the cache. When adding a new CPT, add its `*_LIST_QUERY` constant to `src/queries.js` first.
