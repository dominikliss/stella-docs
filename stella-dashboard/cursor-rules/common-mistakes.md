# Cursor rule export: `common-mistakes.mdc`

_Source of agent rule in theme repo: `.cursor/rules/common-mistakes.mdc`._

---

# Common Mistakes — Don't Do These

## JS mistakes

| Mistake | Fix |
|---|---|
| `import { useState } from 'react'` | Use `from '@wordpress/element'` — WordPress provides React |
| `new Date('2026-03-15')` | Use `parseDateLocal('2026-03-15')` from helpers — `new Date` shifts by timezone |
| `useState` + `useEffect` for derived data | Use `useMemo` — avoids extra render cycles |
| `fetch('/dls/v1/...')` | Use `apiFetch({ path: '/dls/v1/...' })` — handles nonce/auth |
| Inline `{ per_page: -1, order: 'asc', ... }` in invalidate | Use named constants from `src/queries.js` — key order must match exactly |
| `alert()` or `console.log` for user feedback | Use `addToast()` from `useToast` |
| Building custom slide-in panel | Use `SidebarForm` component |
| Building custom confirm dialog | Use `DeleteModal` with `description` prop |
| Hardcoding `invalidateResolution` query args | Import from `src/queries.js` (`TRANSACTION_LIST_QUERY`, etc.) |
| Adding `window.location.reload()` after save | Invalidate the entity cache instead — `invalidateResolution('getEntityRecords', ...)` |
| User-facing **„KI“** for artificial intelligence (`KI-Chat`, `E-Mail-KI-…`) | Use **„AI“** — `AI-Chat`, `E-Mail-AI-Analysen`, `AI-Anbindungen`, routes `ai-anbindungen` / `ai-assistent` |

## SCSS mistakes

| Mistake | Fix |
|---|---|
| Editing `assets/css/*.css` | Edit `assets/scss/**/*.scss` — CSS is compiled from SCSS |
| `font-size: 0.85rem` or `font-size: 12px` | Minimum is `1rem` (14px). Use opacity/weight/tracking for hierarchy |
| Raw hex colours (`#1E1E1E`) | Use SCSS tokens (`$black-2`) from `variables.scss` |
| New class that duplicates `.card` or `.entity-panel` | Reuse the existing class — check `reuse-components.mdc` |
| Creating styles for one-off use | Ask the user first — likely an existing class covers the need |

## PHP mistakes

| Mistake | Fix |
|---|---|
| `api/v1` namespace for browser routes | Use `dls/v1` — `api/v1` is IP-restricted to specific servers |
| `update_post_meta` for ACF fields | Use `update_field($field_key, $value, $post_id)` with the ACF field key |
| `get_field()` on `"ui": 1` select fields | Returns array, not scalar — check ACF JSON config for `"ui"` setting |
| Adding `require` for a new service class | Services autoload via Composer classmap — run `composer dump-autoload` |
| Adding `require` for routes without checking `functions.php` order | Insert in the routes block of `functions.php`, following existing grouping |

## Architecture mistakes

| Mistake | Fix |
|---|---|
| Creating a WP page for a new screen | Use virtual app routes in `inc/app-routes.php` |
| Using GraphQL / Apollo references | GraphQL was removed — use REST API via `@wordpress/core-data` |
| Putting business logic in React components | Keep it in PHP services (`inc/services/`) or REST route callbacks |
| Creating new REST endpoints without auth | Always add `'permission_callback' => 'dls_require_login'` or feature-specific callback |

## Documentation mistakes

| Mistake | Fix |
|---|---|
| Adding long guides under theme `docs/*.md` | Put them in **`docs/stella-docs/`** (`stella-dashboard/`, `stella-server/`, `integration/`) — see **`documentation-source-of-truth.mdc`** |
| Changing behaviour / REST / options / cron without touching docs | Update the matching markdown in **`docs/stella-docs/`**; commit submodule + bump gitlink in theme |
| Editing `.mdc` substantively but not the export | Update **`docs/stella-docs/stella-dashboard/cursor-rules/<same-name>.md`** |
| Duplicating full architecture in `.mdc` | Keep **`architecture.md`** in submodule as narrative; **`architecture.mdc`** stays a short pointer |
