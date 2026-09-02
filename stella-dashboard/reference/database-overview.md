# ddashboard — custom database tables (overview)

All table names below use the WordPress table prefix **`$wpdb->prefix`** (typically `wp_`). Schema for every custom table lives under **`inc/Schema/schemas/*.php`** and is applied via **`dbDelta()`** by the central **`DLS\Schema\SchemaRegistry`** (registered in `inc/Schema/Bootstrap.php`, hooked once on `init` priority 5 from `functions.php`). Version numbers are stored in **`wp_options`** as listed.

Most domain data for invoices, transactions, files, and products lives in **posts** (`wp_posts`) and **post meta**. **Clients and persons** are an exception: they live in dedicated custom tables (see below) while a hidden CPT keeps the historical `wp_posts.ID` reserved for cross-references.

> **Architecture note:** The new `DLS\Domain\*` value objects (one `final readonly` class per table) and the `DLS\Persistence\Repository` base offer typed CRUD over these tables. See **[`../architecture.md`](../architecture.md) → "PHP layers — Domain / Persistence / Schema"** for the full layering. Existing `*DbService` classes are still in place; future PRs should move them onto `Repository<Model>`.

---

## Clients + Persons (WP-independent)

| | |
|---|---|
| **Schema** | `inc/Schema/schemas/ClientPersonSchema.php` |
| **Version** | `dls_client_person_db_version` — `DLS_CLIENT_PERSON_DB_VERSION` |
| **Models** | `DLS\Domain\Client`, `DLS\Domain\Person`, `DLS\Domain\ClientPerson`, `DLS\Domain\ClientInvoiceEmail`, `DLS\Domain\ClientSocial`, `DLS\Domain\ClientReferralReport` |
| **Tables** | `dls_client`, `dls_person`, `dls_client_person` (junction), `dls_client_invoice_email`, `dls_client_social`, `dls_client_referral_report` |

- **ID continuity**: `dls_client.id` and `dls_person.id` reuse the historical `wp_posts.ID` from the migrated CPT entries so existing references (`file.client_id`, `invoice.client_id`, `usermeta.client`, etc.) keep resolving without renumbering. The CPT → custom-table backfill has been completed and finalized in production; the one-shot `ClientPersonMigrationService` and `dls_client_person_finalize_autoincrement()` helper were removed.
- **REST**: `/wp/v2/client` and `/wp/v2/person` are served by `ClientRestController` / `PersonRestController` (bound via `rest_controller_class` on the hidden CPT). Native table-backed CRUD also exposed under `dls/v1/clients` and `dls/v1/persons` for tools/scripts that don't need `@wordpress/core-data`.
- **Services**: `ClientDbService`, `PersonDbService` (CRUD + REST envelope); `ClientService` is a thin legacy adapter returning `stdClass` to PDF / KSeF / Saldeo / file consumers. Fresh staging clones must be restored from a production database snapshot.

---

## Mail (IMAP v3)

| | |
|---|---|
| **Schema** | `inc/Schema/schemas/MailSchema.php` |
| **Version** | Option `dls_mail_db_version` — constant `DLS_MAIL_DB_VERSION` |
| **Models** | `DLS\Domain\MailAccount`, `MailFolder`, `MailMessage`, `MailFolderLink`, `MailMessageLink`, `MailAttachment`, `MailClassificationRule`, `MailSyncRun`, `MailStellaIndexRun` |
| **Tables** | `dls_mail_account`, `dls_mail_folder`, `dls_mail_message`, `dls_mail_folder_link`, `dls_mail_message_link`, `dls_mail_attachment`, `dls_mail_classification_rule`, `dls_mail_sync_run`, `dls_mail_stella_index_run` |

The schema's `beforeUpgrade` callback drops the legacy `dls_mailbox` / `dls_email` tables and the `client_id` column from `dls_mail_message` on early upgrades. **Docs:** [`mail-nachrichten.md`](../mail-nachrichten.md)

---

## Bank accounts

| | |
|---|---|
| **Schema** | `inc/Schema/schemas/BankAccountSchema.php` |
| **Version** | `dls_bank_account_db_version` — `DLS_BANK_ACCOUNT_DB_VERSION` |
| **Model** | `DLS\Domain\BankAccount` |
| **Tables** | `dls_bank_account` |

`afterInstall` seeds default accounts on a fresh install only. Used for invoice PDF / KSeF bank selection (see `BankAccountDbService`).

---

## Project management (PM)

| | |
|---|---|
| **Schema** | `inc/Schema/schemas/PmSchema.php` |
| **Version** | `dls_pm_db_version` — `DLS_PM_DB_VERSION` |
| **Models** | `DLS\Domain\PmProject`, `PmTaskList`, `PmTask`, `PmTaskAssignee`, `PmComment`, `PmTimeEntry` |
| **Tables** | `dls_pm_project`, `dls_pm_task_list`, `dls_pm_task`, `dls_pm_task_assignee`, `dls_pm_comment`, `dls_pm_time_entry` |

`beforeUpgrade` renames the legacy `client_post_id` column to `client_id` for installs upgrading from version 1. REST: `/dls/v1/pm/…` — see OpenAPI export.

---

## YouTube

| | |
|---|---|
| **Schema** | `inc/Schema/schemas/YoutubeSchema.php` |
| **Version** | `dls_youtube_db_version` — `DLS_YOUTUBE_DB_VERSION` |
| **Models** | `DLS\Domain\YoutubeChannel`, `YoutubeVideo`, `YoutubeSyncRun`, `YoutubeAnalyticsSyncRun`, `YoutubeAnalyticsChunk` |
| **Tables** | `dls_youtube_channel`, `dls_youtube_videos`, `dls_youtube_sync_runs`, `dls_youtube_analytics_sync_runs`, `dls_youtube_analytics_chunks` |

---

## AI chat (Claude / multi-provider UI)

| | |
|---|---|
| **Schema** | `inc/Schema/schemas/AiChatSchema.php` |
| **Version** | `dls_ai_chat_db_version` — `DLS_AI_CHAT_DB_VERSION` |
| **Models** | `DLS\Domain\AiChatSession`, `AiChatMessage` |
| **Tables** | `dls_ai_chat_session`, `dls_ai_chat_message` |

---

## AI layers (connectors, analysis configs, actions)

| | |
|---|---|
| **Schema** | `inc/Schema/schemas/AiLayersSchema.php` |
| **Version** | `dls_ai_layers_db_version` — `DLS_AI_LAYERS_DB_VERSION` |
| **Models** | `DLS\Domain\AiConnector`, `AiAnalysisConfig`, `AiAction` |
| **Tables** | `dls_ai_connector`, `dls_ai_analysis_config`, `dls_ai_action` |

Legacy default-connector / default-analysis seed helpers were removed from the schema definition; seeding is no longer performed automatically.

---

## AI profiles (Ollama corpus runs, Verwaltung)

| | |
|---|---|
| **Schema** | `inc/Schema/schemas/AiProfileSchema.php` |
| **Version** | `dls_ai_profile_db_version` — `DLS_AI_PROFILE_DB_VERSION` |
| **Models** | `DLS\Domain\AiProfile`, `AiProfileRun` |
| **Tables** | `dls_ai_profile`, `dls_ai_profile_run` |

The legacy `ai-profile-seed-prompts.php` helper and one-shot writing-style → AI-profile run history migration were removed. Production data has been migrated; fresh installs start with no seeded profiles.

---

## Writing-style history (legacy, read-only)

| | |
|---|---|
| **Schema** | `inc/Schema/schemas/WritingStyleHistorySchema.php` |
| **Version** | `dls_writing_style_history_db_version` — `DLS_WRITING_STYLE_HISTORY_DB_VERSION` |
| **Model** | `DLS\Domain\WritingStyleRun` |
| **Tables** | `dls_writing_style_run` |

Append-only history rows from the previous writing-style flow. Active writes go to `dls_ai_profile_run`; this table is kept for historical lookups only.

---

## Not stored in custom tables

- **Commission PDFs (referrals):** file storage under uploads / commission helper — see `inc/commission-reports-storage.php` (not a dedicated `dls_*` table).
- **KSeF, Saldeo, TrackingTime, NBP:** external APIs; credentials and options in `wp_options`, not separate tables.
- **Expense receipt imports:** files and transaction links use existing CPT/meta patterns plus any storage path from `ExpenseReceiptStorageService` — see theme service when extending.

---

## Schema changes

When you **`ALTER`** or add tables:

1. Edit the relevant `inc/Schema/schemas/*Schema.php` (or add a new schema file + register it in `inc/Schema/Bootstrap.php`).
2. Bump the schema's target version constant; add idempotent `beforeUpgrade` logic only if `dbDelta` cannot do the change cleanly.
3. Add or update the matching `DLS\Domain\*` model and any `Repository` subclass column list.
4. Update this file and `architecture.md` if the table list or purpose changes.
5. Follow the theme workflow in `documentation-source-of-truth` (submodule + architecture).
