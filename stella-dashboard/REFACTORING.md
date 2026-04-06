# Refactoring Plan — Remaining Steps

This document tracks what was done in the March 2026 cleanup and what's left.

## Completed

### Phase 1 — Dead Code & Quick Wins
- [x] Removed dead code from functions.php (debug block, unused $file_service, noop REST filter)
- [x] Removed pointless self-instantiations from 4 service files
- [x] Created inc/constants.php (DLS_ALLOWED_IPS, DLS_OWNER_EMAILS, DLS_OWNER_DOMAINS, DLS_ANALYSIS_TIMEOUT)
- [x] Added dls_require_login() and adopted across 15 route files (30 replacements)
- [x] Standardized index.js to side-effect-only imports

### Phase 2 — God File Splits
- [x] helpers.js → barrel + 6 sub-modules (currency, dates, data-utils, acf-normalize, rest, contact)
- [x] management-page.js (1896→95 lines) → sub-components (ksef-settings, tracking-time-settings, youtube-settings, ai-provider-settings); E-Mail-AI UI später als `ai-profiles-page.js` (Verwaltung → AI-Profile)
- [x] mail-admin-tab.js (2012→946 lines) → 4 sub-components (mailbox-imap-status, mailbox-category-manager, mailbox-folder-detail, mailbox-sync-controls)
- [x] mailbox-db-service.php (1575→380 lines) → 3 focused services (EmailCrudService, EmailSpamService, EmailCategoryDbService)
- [x] file-service.php (1574→818 lines) → 2 focused services (InvoicePdfService, CommissionReportService)
- [x] mailboxes.php (1349→410 lines) → 4 route files (mail-emails, mail-spam, mail-categories, mail-sync)
- [x] messages.scss (1505→10 lines) → 3 partials (mail-admin-panel, inbox, chunk-sync)

### Phase 3 — Consistency & Compliance
- [x] Fixed 112 sub-1rem font-size violations across 20 SCSS files
- [x] Added SCSS design tokens (spacing scale, border radius, shadows)
- [x] Standardized useDispatch to store handle pattern (3 files)
- [x] Removed axios dependency, migrated to @wordpress/api-fetch
- [x] Unified PHP namespace (DlData → DLS\Services)

### Phase 4 — Infrastructure
- [x] Composer classmap autoloading for services (removed 38 manual requires)
- [x] Jest test harness with 28 tests covering helper functions
- [x] Created useRestAction hook (src/hooks/use-rest-action.js)

---

## Remaining / Recommended Next Steps

### High Priority — Smoke Test
- [ ] **Test in browser**: Verwaltung tabs (KSeF, TrackingTime, YouTube, AI, Email Analysis)
- [ ] **Test in browser**: Nachrichten inbox, mailbox admin, sync controls
- [ ] **Test in browser**: Accounting page, transaction CRUD, invoices
- [ ] **Test in browser**: Client form, file form, PDF generation
- [ ] **Test**: Portal user create/delete flow
- [ ] **Test**: Login form (uses migrated apiFetch instead of axios)

### Medium Priority — Adopt New Patterns
- [ ] Adopt `useRestAction` hook in components that have the `apiFetch + try/catch + addToast` pattern
  - Start with: transaction-form.js, client-form.js, file-form.js, invoice-form.js
  - Then: project-management.js, messages-page.js, chat-page.js
- [ ] Start using SCSS spacing tokens ($space-xs through $space-3xl) when touching style files
- [ ] Start using SCSS shadow tokens ($shadow-card, $shadow-dropdown, $scrim) to replace raw rgba values
- [ ] Start using border radius tokens ($radius-sm, $radius-md, $radius-pill) when touching style files
- [ ] Replace raw hex/rgba color values with $-variables (especially in formik-date-picker.scss)

### Lower Priority — Further Refactoring
- [ ] Break up remaining large PHP methods (200+ lines) in MailChunkSyncService, SubscriptionBillingService
- [ ] Split accounting-stats.js (584 lines) — separate computation from React mount
- [ ] Split chat-page.js (752 lines) — separate ChatComposer, ChatStartScreen, ChatConversationView into own files
- [ ] Split formik-date-picker.scss (827 lines) — separate library overrides from theme layer
- [ ] Reduce SCSS nesting depth (4-6 levels → max 3) in form.scss, conversation-layout.scss
- [ ] Add more Jest tests: REST error handling, contact helpers, data-utils
- [ ] Add PHPUnit tests: SubscriptionBillingService, ClientEmailMatchService, NbpExchangeRateService
- [ ] Clean up cursor rules that overlap (architecture.mdc vs mail-nachrichten.mdc)
- [ ] Remove h7/h8 dead selectors from styles.scss

### Infrastructure
- [ ] Move @wordpress/scripts from dependencies to devDependencies in package.json
- [ ] Consider code splitting (dynamic import) to reduce bundle size (currently 1.12 MiB)
- [ ] Update browserslist database (caniuse-lite is 10 months old)
- [ ] Consider renaming service files to PascalCase for true PSR-4 (currently using classmap due to kebab-case)
