# Cursor rule export: `english-code-naming.mdc`

_Source of agent rule in theme repo: `.cursor/rules/english-code-naming.mdc`._

---

# English naming (code & files)

All **developer-facing** names MUST be in **English**. This is an **always-on** convention: filenames, identifiers, and structural markup hooks used in code — not user-facing copy.

## Applies to

- **File and folder names** — e.g. `admin-settings-page.js`, `mail-admin-tab.js`, `messages-focus-inbox-row.js`; not `verwaltung-page.js`, `nachrichten-focus-inbox-row.js`, or German path segments.
- **React components** — `function MailAdminTab` / `export default function AdminSettingsPage`, not German component or prop *names* (prop values shown to users may be German).
- **Hooks, helpers, modules** — `useToast`, `formatRestSaveError`, `mailPath`; no German function, variable, parameter, or `const` names (except literals mirroring external APIs).
- **PHP (theme code)** — class names, methods, `function dls_*` where the suffix describes behaviour in English; file names under `inc/` in English (`mail-admin-tab.php` pattern). Existing `dls_*` hooks/options that are already deployed may keep their slugs until a deliberate migration.
- **SCSS** — English partial file names (`_admin-settings.scss`); **variables, mixins, placeholders, function names** in English (e.g. `$black-3`, not `$fokus-rand`). **BEM / block classes** for new UI in English (`dls-messages-page__*`). Legacy classes tied to PHP templates may stay until a coordinated rename.
- **HTML hooks in app code** — Prefer English `id` / `data-*` / `class` segments for new work (`dls-inbox-email-search`). Exception: stable mount points or WP body classes already in production (e.g. `.dls-nachrichten`) until an explicit migration.

## Does not replace product locale

- **User-visible strings** (UI labels, toasts, `__()`, `esc_html_e()`, email copy) stay **German** when the product is German.
- **URL paths** already in use (`/verwaltung/…`, `/nachrichten`) are product/SEO decisions; do not rename for “English only” without an explicit migration task.

## When touching German-named legacy files

Prefer **renaming to English** as part of the same change (update imports, PHP `require`, webpack entry if any, and grep for old paths). If a rename is out of scope, still use **English** for any **new** symbols added in that file.

## Examples

| Avoid | Prefer |
|-------|--------|
| `VerwaltungPage`, `verwaltung-page.js` | `AdminSettingsPage`, `admin-settings-page.js` |
| `NachrichtenInbox`, `nachrichten-focus-inbox-row.js` | `MessagesInbox`, `messages-focus-inbox-row.js` |
| `const postfachListe = …` | `const mailboxList = …` |

Use one clear English term per concept across PHP and JS (e.g. `mail`, `mailbox`, `message` — pick the domain term already dominant in the repo for that feature).
