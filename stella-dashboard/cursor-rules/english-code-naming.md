# Cursor rule export: `english-code-naming.mdc`

_Source of agent rule in theme repo: `.cursor/rules/english-code-naming.mdc`._

---

# English naming (code & files)

All **developer-facing** names MUST be in **English**. This keeps the codebase consistent and searchable for mixed-language teams and tooling.

## Applies to

- **File and folder names** — e.g. `admin-settings-page.js`, `mail-admin-tab.js`, not `verwaltung-page.js` / `mail-verwaltung-tab.js`.
- **React components** — `function MailAdminTab` / `export default function AdminSettingsPage`, not German component identifiers.
- **Hooks, helpers, modules** — `useToast`, `formatRestSaveError`, `mailPath`; no German function or variable names in source (except where mirroring external APIs).
- **PHP (theme code)** — class names, methods, `function dls_*` where the suffix describes behaviour in English; file names under `inc/` in English (`mail-admin-tab.php` pattern). Existing `dls_*` hooks/options that are already deployed may keep their slugs until a deliberate migration.
- **SCSS partials** — English file names (`_admin-settings.scss`); BEM or block names in English unless tied 1:1 to a documented WP/class hook that must stay fixed.

## Does not replace product locale

- **User-visible strings** (UI labels, toasts, `__()`, `esc_html_e()`, email copy) stay **German** when the product is German.
- **URL paths** already in use (`/verwaltung/…`, `/nachrichten`) are product/SEO decisions; do not rename for “English only” without an explicit migration task.

## When touching German-named legacy files

Prefer **renaming to English** as part of the same change (update imports, PHP `require`, webpack entry if any, and grep for old paths). If a rename is out of scope, still use **English** for any **new** symbols added in that file.

## Examples

| Avoid | Prefer |
|-------|--------|
| `VerwaltungPage`, `verwaltung-page.js` | `AdminSettingsPage`, `admin-settings-page.js` |
| `NachrichtenInbox` (identifier) | `MessagesInbox` or `MailInbox` |
| `const postfachListe = …` | `const mailboxList = …` |

Use one clear English term per concept across PHP and JS (e.g. `mail`, `mailbox`, `message` — pick the domain term already dominant in the repo for that feature).
