# ddashboard — custom database tables (overview)

All table names below use the WordPress table **`$wpdb->prefix`** (typically `wp_`). The theme creates or upgrades these via **`dbDelta()`** in `inc/install-*.php`. Version numbers are stored in **`wp_options`** as listed.

Domain data for clients, invoices, transactions, etc. also lives in **posts** (`wp_posts`) and **post meta** — this document lists **only custom tables** added by the theme.

---

## Mail (IMAP v3)

| | |
|---|---|
| **Install** | `inc/install-mail-tables.php` |
| **Version** | Option `dls_mail_db_version` — constant `DLS_MAIL_DB_VERSION` (see file) |
| **Tables** | `dls_mail_account`, `dls_mail_folder`, `dls_mail_message`, `dls_mail_folder_link`, `dls_mail_message_link`, `dls_mail_attachment`, `dls_mail_classification_rule`, `dls_mail_sync_run` |

**Docs:** [`mail-nachrichten.md`](../mail-nachrichten.md)

---

## Bank accounts

| | |
|---|---|
| **Install** | `inc/install-bank-account-tables.php` |
| **Version** | `dls_bank_account_db_version` — `DLS_BANK_ACCOUNT_DB_VERSION` |
| **Tables** | `dls_bank_account` |

Used for invoice PDF / KSeF bank selection (see `BankAccountDbService`).

---

## Project management (PM)

| | |
|---|---|
| **Install** | `inc/install-pm-tables.php` |
| **Version** | `dls_pm_db_version` — `DLS_PM_DB_VERSION` |
| **Tables** | `dls_pm_project`, `dls_pm_task_list`, `dls_pm_task`, `dls_pm_task_assignee`, `dls_pm_comment`, `dls_pm_time_entry` |

REST: `/dls/v1/pm/…` — see OpenAPI export.

---

## YouTube

| | |
|---|---|
| **Install** | `inc/install-youtube-tables.php` |
| **Version** | `dls_youtube_db_version` — `DLS_YOUTUBE_DB_VERSION` |
| **Tables** | `dls_youtube_channel`, `dls_youtube_videos`, `dls_youtube_sync_runs`, `dls_youtube_analytics_sync_runs`, `dls_youtube_analytics_chunks` |

---

## AI chat (Claude / multi-provider UI)

| | |
|---|---|
| **Install** | `inc/install-ai-chat-tables.php` |
| **Version** | `dls_ai_chat_db_version` — `DLS_AI_CHAT_DB_VERSION` |
| **Tables** | `dls_ai_chat_session`, `dls_ai_chat_message` |

---

## AI layers (connectors, analysis configs, actions)

| | |
|---|---|
| **Install** | `inc/install-ai-layers-tables.php` |
| **Version** | `dls_ai_layers_db_version` — `DLS_AI_LAYERS_DB_VERSION` |
| **Tables** | `dls_ai_connector`, `dls_ai_analysis_config`, `dls_ai_action` |

---

## AI profiles (Ollama corpus runs, Verwaltung)

| | |
|---|---|
| **Install** | `inc/install-ai-profile-tables.php` |
| **Version** | `dls_ai_profile_db_version` — `DLS_AI_PROFILE_DB_VERSION` |
| **Tables** | `dls_ai_profile`, `dls_ai_profile_run` |

---

## Writing-style history (legacy + unified history rows)

| | |
|---|---|
| **Install** | `inc/install-writing-style-history.php` |
| **Version** | `dls_writing_style_history_db_version` — `DLS_WRITING_STYLE_HISTORY_DB_VERSION` |
| **Tables** | `dls_writing_style_run` |

Used by writing-style flows and related AI history listings; see `ai-writing-style.php` and AI profile migration notes in `install-ai-profile-tables.php`.

---

## Not stored in custom tables

- **Commission PDFs (referrals):** file storage under uploads / commission helper — see `inc/commission-reports-storage.php` (not a dedicated `dls_*` table).
- **KSeF, Saldeo, TrackingTime, NBP:** external APIs; credentials and options in `wp_options`, not separate tables.
- **Expense receipt imports:** files and transaction links use existing CPT/meta patterns plus any storage path from `ExpenseReceiptStorageService` — see theme service when extending.

---

## Schema changes

When you **`ALTER`** or add tables:

1. Bump the relevant `DLS_*_DB_VERSION` and option check in the matching `install-*.php`.
2. Update this file if the table list or purpose changes.
3. Follow the theme workflow in `documentation-source-of-truth` (submodule + architecture).
