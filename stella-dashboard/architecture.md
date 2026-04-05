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
- `DLS\Services\` — all services (`FileService`, `ClientService`, `OptionService`, `BasicService`, `KSeFService`, `SaldeoExportService`, `SubscriptionBillingService`, `YouTubeDataService`, `YouTubeSyncService`, `YouTubeOAuthService`, `YouTubeAnalyticsApiService`, `YouTubeAnalyticsSyncService`, `AiAgentRegistry`, `AnthropicChatService`, `AiChatSessionService`, `PmDbService`, `MailboxDbService`, `EmailCrudService`, `EmailSpamService`, `EmailCategoryDbService`, `MailSyncService`, `MailChunkSyncService`, `ClientEmailMatchService`, `MailThreadIdService`, `MailHtmlSplitQuoteService`, `MailAttachmentStorageService`, `MailCrypto`, `DlsImapFactory`, `WritingStyleHistoryService`, `NbpExchangeRateService`, `InvoicePdfService`, `CommissionReportService`). Services are autoloaded via Composer classmap.
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
  constants.php              # App-wide constants: DLS_ALLOWED_IPS, DLS_OWNER_EMAILS, DLS_OWNER_DOMAINS, DLS_ANALYSIS_TIMEOUT
  subscription-billing.php   # Clears legacy subscription cron; billing is manual via REST only
  mail-sync-cron.php         # WP-Cron: `dls_mail_imap_sync_cron` every 5 min → sync active mailboxes
  install-mail-tables.php    # dbDelta: dls_mailbox, dls_email, spam lists, folder_client, dls_email_attachment; migrations v1–v8
  install-pm-tables.php      # dbDelta: dls_pm_project, dls_pm_task_list, dls_pm_task, dls_pm_task_assignee, dls_pm_comment, dls_pm_time_entry; migrations v1–v2
  install-ai-chat-tables.php # dbDelta: dls_ai_chat_session, dls_ai_chat_message
  install-writing-style-history.php # dbDelta: dls_writing_style_run + analysis_type (v2); append-only email AI analysis history
  install-youtube-tables.php # dbDelta: YouTube Data + Analytics custom tables
  acf/settings/              # ACF field group JSON configs (group_*.json)
  post-types/                # CPT registration
  routes/                    # REST API endpoints (dls/v1 namespace unless noted)
    mailboxes.php            # Mailbox CRUD, folder-client mappings, folder-clients-overview
    mail-emails.php          # Email list/get/update + attachment endpoints (split from mailboxes.php)
    mail-spam.php            # Spam confirm, blocklist, whitelist endpoints (split from mailboxes.php)
    mail-categories.php      # Email category CRUD endpoints (split from mailboxes.php)
    mail-sync.php            # Sync, chunk-sync, recompute, clear-client-links (split from mailboxes.php)
    pm-projects.php          # /dls/v1/pm/* — projects, task lists, tasks, assignees, comments, time entries
    tracking-time.php        # TrackingTime Pro import (time entries, tasks, projects)
    ksef-import.php          # POST /dls/v1/ksef-import
    ksef-test-connection.php # POST /dls/v1/ksef-test-connection
    youtube-sync.php         # POST /dls/v1/youtube/sync, GET /dls/v1/youtube/status
    youtube-analytics.php    # OAuth + YouTube Analytics API sync → dls_youtube_analytics_*
    saldeo-export.php        # GET  /dls/v1/saldeo-export
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
    mailbox-db-service.php   # MailboxDbService facade — mailbox CRUD + delegates to EmailCrudService, EmailSpamService, EmailCategoryDbService
    email-crud-service.php   # EmailCrudService — email record CRUD (split from MailboxDbService)
    email-spam-service.php   # EmailSpamService — spam heuristics, blocklist/whitelist (split from MailboxDbService)
    email-category-db-service.php # EmailCategoryDbService — email category CRUD (split from MailboxDbService)
    mail-sync-service.php    # IMAP import (newest-N fetch per folder); date_sent fallback; prune_emails_removed_from_imap_folder (full UID map vs DB)
    mail-chunk-sync-service.php # Background chunked IMAP sync; prune_imap_orphans_for_completed_chunk_job on job complete
    client-email-match-service.php # resolve_client_id_single_counterparty_exact (strict) + resolve_client_id (legacy)
    mail-thread-id-service.php
    mail-html-split-quote-service.php # split_latest_and_quoted — strips quoted history from body_html for list
    mail-attachment-storage-service.php # store IMAP attachments → wp-content/private/dls-mail-attachments/
    mail-crypto.php          # Mailbox password AES encryption
    dls-imap-factory.php     # Webklex PHPIMAP client factory (no ext-imap)
    pm-db-service.php        # PmDbService: projects / task lists / tasks / assignees / comments / time entries
    ksef-service.php         # KSeF API 2.0 (production only)
    saldeo-export-service.php
    subscription-billing-service.php
    nbp-exchange-rate-service.php
    option-service.php       # WP options: ksef_*, saldeo_*, youtube_*, tracking_time_pro_*
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
    invoice-pdf-service.php  # InvoicePdfService — invoice PDF generation (split from FileService)
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
    accounting.js            # Main dashboard — PageLayout (filter + actions), chart, transaction list, KSeF import
    transactions.js          # Reusable transaction table with expand/edit/delete (uses EntityListTable)
    transaction-form.js      # Formik form for creating/editing transactions (uses useFormSubmit)
    transaction-card.js      # Expanded detail view for a single transaction (uses DataTable + CrudActions)
    sidebar-form.js          # Slide-in panel; optional `wide`, `className` (e.g. sidebar-form--template-edit)
    file-form.js             # Formik form for file CPT (TipTap, template modes)
    tiptap-editor.js         # TipTap StarterKit wrapper for file body HTML
    expanding-content.js     # Animated height expand/collapse
    invoices.js / invoice-form.js  # Invoices list uses PageLayout + EntityListTable + useEntityCrud; detail uses DataTable
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
    messages-page.js         # /nachrichten inbox — threads, spam, blocklist/whitelist (uses useDlsQuery + useRestAction)
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
    mail-admin-tab.js        # Mailbox admin orchestrator: IMAP settings, folder-client mapping, sync buttons, clear-client-links
    mailbox-imap-status.js   # IMAP status presentation helpers (split from mail-admin-tab.js)
    mailbox-category-manager.js # Email category CRUD UI (split from mail-admin-tab.js)
    mailbox-folder-detail.js # Folder-client mapping UI (split from mail-admin-tab.js)
    mailbox-sync-controls.js # Sync/chunk-sync UI + useMailboxSync hook (split from mail-admin-tab.js)
    management-page.js       # /verwaltung integrations orchestrator (delegates to sub-components)
    management-integration-card.js # IntegrationServiceCard shell (icon, title, badge, header actions)
    ksef-settings.js         # KSeF config (split from management-page.js)
    tracking-time-settings.js # TrackingTime config (split from management-page.js)
    youtube-settings.js      # YouTube OAuth + config (split from management-page.js)
    ai-provider-settings.js  # Anthropic/Ollama config (split from management-page.js)
    email-analysis-panel.js  # Writing style + classification analysis UI (split from management-page.js)
    admin-settings-page.js   # Mounts AdminSettingsPage: tabs buchhaltung/projekte/marketing/ai-anbindungen/ai-assistent/nachrichten; AI-Anbindungen + AI-Analyse eigene Seiten
    marketing.js             # Marketing module (/marketing/start, /marketing/youtube)
    ai-chat-panel.js         # Claude chat UI (agentKey prop)
    master-detail-layout.js  # MasterDetailLayout — two-column master/detail generic wrapper
    scroll-panel.js          # ScrollPanel — scrollable flex child with optional header/footer slots
    list-search-field.js     # Debounced list search field
    table-pagination.js      # TablePagination — server-paginated numeric pager (DEFAULT_TABLE_PAGE_SIZE=200)
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
- `/verwaltung` has sub-routes: `buchhaltung`, `projekte`, `marketing`, `ai-anbindungen`, `ai-assistent`, `nachrichten` (tab selection via `data-subroute`).
- `/marketing` has sub-routes: `start` (default; `/marketing/` redirects), `youtube` — PHP `.sub-header` tabs + `.site-meta` h1 + `data-subroute` on `.dls-marketing` (same shell as Buchhaltung + Verwaltung tabs).
- `/werkzeuge` has sub-routes: `email-migration` (default; `/werkzeuge/` opens it) — PHP `.sub-header` + `.site-meta` h1 + `tools-page.js` on `.dls-werkzeuge` (`data-subroute`). Nav link in `header.php`.
- `AdminSettingsPage` in `admin-settings-page.js`: switches between `MailAdminMailboxes` (nachrichten), `AiConnectorsPage` / `AiAnalysisActionsPage` (AI tabs), and `ManagementPage` (buchhaltung/projekte/marketing).
- Add new screens: add to `dls_get_app_routes()`, add `add_rewrite_rule`, add to `admin-settings-page.js` tab list if it's a Verwaltung tab, mount React component. Flush permalinks after adding.
- **Do not** create a WP page with the same slug.

## Nachrichten (IMAP)

- **Tables** (`inc/install-mail-tables.php`, option `dls_mail_db_version`): `dls_mailbox`, `dls_email` (`spam_status` 0 / 1 / 2), `dls_email_spam_blocklist`, `dls_email_spam_whitelist`, `dls_mailbox_folder_client`, `dls_email_attachment` (attachment bytes under `wp-content/private/dls-mail-attachments/`).
- **Services:** `MailboxDbService` (facade), `EmailCrudService`, `EmailSpamService`, `EmailCategoryDbService`, `MailSyncService`, `MailChunkSyncService`, `ClientEmailMatchService`, `MailThreadIdService`, `MailHtmlSplitQuoteService`, `MailAttachmentStorageService`, `MailCrypto`, `DlsImapFactory` — under `inc/services/`.
- **WP-Cron:** `inc/mail-sync-cron.php` — hook `dls_mail_imap_sync_cron` every **5 minutes** (`dls_every_five_minutes` custom schedule); syncs all active mailboxes (limit 50 each, filterable via `dls_mail_cron_sync_limit`).
- **Import vs. prune:** Folder import only **fetches the newest N** messages per run (`limit` / cron limit). **Orphan prune** is separate: after sync, `MailSyncService::prune_emails_removed_from_imap_folder()` compares DB `imap_uid` rows for that folder to the **full current UID set** from IMAP (`getUid()->data()`). Messages still on the server but outside the N-batch are **not** deleted. Rows with NULL `imap_uid` are not UID-pruned. Chunk sync runs the same prune when a background job completes (`MailChunkSyncService::prune_imap_orphans_for_completed_chunk_job`). Sync `debug` may include `imap_pruned`. **UIDVALIDITY** changes remain the standard IMAP caveat.
- **REST:** `inc/routes/mailboxes.php` (mailbox CRUD, folder-client mappings), `mail-emails.php` (emails list/get/update, attachments), `mail-spam.php` (spam confirm, blocklist, whitelist), `mail-categories.php` (email categories), `mail-sync.php` (sync, chunk-sync, recompute, clear-client-links) — all `dls/v1`.
- **Spam on sync:** Blocklist wins (2); whitelist forces 0 and skips heuristics; else emoji-in-subject and subdomain-sender heuristics (with dominikliss/foxcraft + client-website exemptions) — `EmailSpamService::resolve_email_spam_status` / `EmailCrudService::upsert_email_row`.
- **Client auto-assign (new strict rules, 2025-03):**
  1. **Folder mapping first** (`dls_mailbox_folder_client`) — sets `client_id` before address logic; never overwritten by address logic.
  2. **Address rule:** collect **To, Cc, Bcc, From** only (no Reply-To). Drop: mailbox login, dominikliss.com + subdomains, foxcraft.digital + subdomains, `liss.dominik@gmail.com` / `liss.dominikliss@gmail.com`.
  3. **Exactly one** distinct external address → assign if it matches a client/invoice/person email.
  4. **More than one** distinct external address → assign only if **all** map to the **same** client in the CRM index.
  5. Otherwise leave `client_id` null. **No domain-from-address, no subject/website domain heuristics, no outbound thread inheritance, no folder consensus propagation.**
  - Implemented in `ClientEmailMatchService::resolve_client_id_single_counterparty_exact()`. Legacy `resolve_client_id()` kept for non-import callers only.
  - `EmailCrudService::clear_all_email_client_links( ?int $mailbox_id )` — bulk-remove `client_id` from `dls_email` rows. Exposed as `POST /dls/v1/emails/clear-client-links` with optional `{ mailbox_id }` body.
  - Verwaltung UI: "Zuordnungen löschen" button per mailbox with confirmation modal (`mail-admin-tab.js`).
- **`date_sent` fallback chain:** `getDate()` → raw `Date:` header scan → iCalendar body parsing (DTSTAMP / CREATED / LAST-MODIFIED / DTSTART in text body, HTML, attachments, or raw MIME body) for calendar/event mails missing a `Date:` header.
- **Quote splitting:** List endpoint strips quoted reply history from `body_html` via `MailHtmlSplitQuoteService::split_latest_and_quoted()`; full body available from `GET /dls/v1/emails/{id}` (`body_has_quote_history` flag on list items).
- **UI:** `messages-page.js` (inbox / conversation view), `mail-admin-tab.js` (orchestrator → `mailbox-imap-status.js`, `mailbox-category-manager.js`, `mailbox-folder-detail.js`, `mailbox-sync-controls.js`), `list-search-field.js` (debounced search). Layout: `MasterDetailLayout` + `ScrollPanel` + `ConversationListItem` + `EmailConversationThread` + `EmailMessageBlock` + `EmailDetailSidebar` + `EmailHtmlIframe` + `MessageAttachmentList`. Filter bar SCSS: `assets/scss/components/messages.scss` (imports `inbox.scss`, `mail-admin-panel.scss`, `chunk-sync.scss`) + `filter-bar.scss`; layout SCSS: `assets/scss/components/conversation-layout.scss`.

## E-Mail-AI-Analysen (Ollama, Verwaltung → AI-Analyse)

Zwei asynchrone Langläufer in **einer** Karte **„E-Mail-AI-Analysen“** auf dem Tab **AI-Analyse** (`ai-analysis-actions-page.js` → `EmailAnalysisPanel`, `variant="embedded"`): **Schreibstil** vs **Klassifizierung** (Select **Analyse-Typ**, globale `field-row` / `field-container` / `select`-Styles). Es darf **nur eine** Analyse gleichzeitig laufen — der jeweils andere Start liefert **409** (`analysis_busy`). **Anbindungen** (Anthropic/Stella) liegen auf **AI-Anbindungen**; Metadaten in `dls_ai_connector`, `dls_ai_analysis_config`, `dls_ai_action` + REST `GET/PATCH /dls/v1/ai/layers/*`.

### Gemeinsame Historie (DB)

- **Tabelle:** `dls_writing_style_run` — Spalte **`analysis_type`**: `writing_style` \| `classification_suggestions` (Konstanten `WritingStyleHistoryService::ANALYSIS_*`). **`profile_text`** = vollständiger Text der letzten erfolgreichen Ausgabe (Markdown-Taxonomie oder Schreibstil-Profil).
- **Install:** `inc/install-writing-style-history.php` — DB-Version inkl. `analysis_type` + Index.
- **Service:** `WritingStyleHistoryService::log_success` / `log_error` — optionaler 5. Parameter `$analysis_type` (Default `writing_style`); Laufzeit kommt aus `dls_writing_style_started_unix` bzw. `dls_email_classification_started_unix`.
- **REST:** `GET /dls/v1/ai/writing-style-history?limit=&offset=&analysis_type=` — **Default `analysis_type` = `writing_style`** (Abwärtskompatibilität). `GET /dls/v1/ai/writing-style-history/{id}` liefert die Zeile inkl. `analysis_type`.

### Schreibstil (`writing_style`)

- **REST:** `inc/routes/ai-writing-style.php` — `GET /dls/v1/ai/writing-style-profile`, `POST /dls/v1/ai/learn-writing-style`, `POST /dls/v1/ai/stop-writing-style`, `POST /dls/v1/ai/writing-style-final-grounding` (`{ final_grounding }`).
- **Corpus:** `dls_email` **ausgehend** (`direction = outbound`, kein Spam), Limit aus Option / Request.
- **Optionen:** `dls_writing_style_status`, `dls_writing_style_profile`, `dls_writing_style_final_grounding`, Limits/Modell, PID, Cron/CLI — siehe Dateikopf der Route.
- **Worker:** `inc/scripts/learn-writing-style.php` — CLI (`nohup php …`) oder Hook `dls_writing_style_cron_run`.
- **Consumer:** `dls_get_active_writing_style_grounding()` — bevorzugt `final_grounding`, sonst `profile`.

### Klassifizierung / Taxonomie (`classification_suggestions`)

- **REST:** `inc/routes/ai-email-classification-suggestions.php` — `GET /dls/v1/ai/email-classification-profile` (u. a. `json` = Rohausgabe Markdown, `final_grounding` = bearbeiteter Text, `json_parsed` nur wenn der String gültiges JSON ist), `POST /dls/v1/ai/learn-email-classification`, `POST /dls/v1/ai/stop-email-classification`, `POST /dls/v1/ai/email-classification-final-grounding` (`{ final_grounding }`). **`functions.php`** lädt diese Datei neben `ai-writing-style.php`.
- **Corpus:** `dls_email` **eingehend** (`direction = inbound`, kein Spam). Requirements-Check zählt ebenfalls nur inbound.
- **Optionen:** `dls_email_classification_status`, `dls_email_classification_json` (Name historisch — Inhalt ist **Markdown**-Taxonomie), `dls_email_classification_final_grounding` (optional, user-edited; Vorrang vor `json`), `dls_email_classification_*` analog Schreibstil.
- **Worker:** `inc/scripts/learn-email-classification-suggestions.php` — CLI oder `dls_email_classification_cron_run`. Prompt: zweistufige Taxonomie (regelbasierte Muster + PHP-Regelstruktur + modellbasierte Abschnitte); Erfolgs-Validierung: Ausgabe enthält **`### Regelbasierte Muster`**; optional äußerer Markdown-Codefence wird entfernt.

### Frontend

- **`refreshEmailAnalysisJobs`:** lädt **beide** Profile parallel; ein Intervall-Poll solange **irgendein** Job `running` ist; Toasts bei Übergang **running → done/error** pro Typ; Historie pro Typ: `analysis_type=writing_style` bzw. `classification_suggestions`.
- Schreibstil: bearbeitbares finales Grounding speichern nur über **`/ai/writing-style-final-grounding`**, nicht nur lokalen State.
- Klassifizierung: bearbeitbares finales Grounding über **`/ai/email-classification-final-grounding`**. **Consumer:** `dls_get_active_email_classification_grounding()` — bevorzugt `final_grounding`, sonst `dls_email_classification_json`.

## Project Management (PM)

- **Tables** (`inc/install-pm-tables.php`, option `dls_pm_db_version` v2): `dls_pm_project` (name, status, client_id, tt_project_id), `dls_pm_task_list` (project_id, sort_order, tt_task_list_id), `dls_pm_task` (title, status, due_date, sort_order, tt_task_id), `dls_pm_task_assignee` (task_id, user_id), `dls_pm_comment` (task_id, user_id, body, tt_comment_id), `dls_pm_time_entry` (task_id, project_id, user_id, minutes, started_at, tt_time_entry_id).
- **Service:** `DLS\Services\PmDbService` (`inc/services/pm-db-service.php`) — full CRUD for projects, task lists, tasks, assignees, comments, time entries. `get_project_id_by_tt_project_id()` for TrackingTime import upsert.
- **REST:** `inc/routes/pm-projects.php` — namespace `dls/v1`, prefix `/pm`; routes for projects, task lists, tasks, assignees, comments, time entries. Auth: `dls_pm_permission()` (logged-in).
- **UI:** `src/components/project-management.js` — mounts on `.dls-project-management`; full CRUD UI with `SidebarForm`, `SearchableSelect` (client), `DeleteModal`, task lists + tasks inline.
- **Virtual route:** `/projects` → `src/index.js` → `project-management.js`. Also accessible as sub-route `projekte` on `/verwaltung/projekte`.
- **TrackingTime link:** `tt_project_id`, `tt_task_id`, etc. stored for idempotent import from TrackingTime Pro API (`inc/routes/tracking-time.php`).

## KSeF Integration
- Service: `inc/services/ksef-service.php` — class `DLS\Services\KSeFService` (namespace `DLS\Services`)
- **Production only** — base URL: `https://api.ksef.mf.gov.pl/api/v2`. No test environment.
- Auth: 6-step async JWT flow (challenge → RSA-OAEP-SHA256 encrypt → POST `/auth/ksef-token` → poll status → redeem → `accessToken`)
- Import window: always fixed 3-month range (first day of 2 months ago → last day of current month). Frontend date params are ignored.
- Upsert behaviour: **on create** — write all fields including `note`, `type`, `post_title`; **on update** — only refresh financial/date fields (`invoice_date`, `price_net`, `vat_percent`, `currency`) and KSeF identifiers. Never overwrite user-edited `note` or `post_title` on re-import.
- Seller name rules (transaction `note` on **create** only): if seller name starts with `"Good Code"` → note `"Freelancer (Pawel)"`; if it starts with `"KANCELARIA PODATKOWA ANDRZEJ NOWAK SPÓŁKA Z OGRANICZONĄ ODPOWIEDZIALNOŚCIĄ"` → note `"Accounting KPAN"` + invoice number (trimmed).
- `ksef_reference` and `ksef_seller_nip` are **ACF fields** (field keys `field_6829ksef000r1` / `field_6829ksef000n2`) — use `update_field($field_key, $value, $post_id)`, not `update_post_meta`.
- PLN conversion rate is auto-populated via `NbpExchangeRateService::get_rate_for_input_date($issue_date)` and stored in `pln_conversion_rate_invoice_date`.
- Settings stored as WP options: `ksef_enabled`, `ksef_nip`, `ksef_token`

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
