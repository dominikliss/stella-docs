# Reference — OpenAPI & database overview

Machine-readable **OpenAPI 3** exports and a **custom MySQL table catalogue** for the ddashboard WordPress theme. Narrative architecture remains in [`architecture.md`](../architecture.md).

## OpenAPI

| File | Namespace | Audience |
|------|-----------|----------|
| [`dls-v1.openapi.json`](dls-v1.openapi.json) | `dls/v1` | Logged-in dashboard app (`apiFetch` + cookie + `X-WP-Nonce`) |
| [`api-v1.openapi.json`](api-v1.openapi.json) | `api/v1` | IP-restricted portal / server integrations only |

**Base URLs (WordPress REST):**

- Dashboard: `{site}/wp-json/dls/v1/…`
- Restricted: `{site}/wp-json/api/v1/…`

These files are **hand-maintained** to match `inc/routes/*.php`. They list paths and HTTP methods with short summaries; they are **not** a full JSON Schema of every request/response body. When you add or change a route, update the JSON (or regenerate via the small Python builder in git history) in the same change series.

**Viewing:** Import into [Swagger Editor](https://editor.swagger.io/), Stoplight, or any OpenAPI viewer. The spec is JSON; YAML is not required.

## Database

| File | Contents |
|------|----------|
| [`database-overview.md`](database-overview.md) | Custom tables created by theme `install-*.php` scripts, version options, and links to services |

Core business data also lives in **WordPress CPTs** (`client`, `transaction`, `invoice`, …) and **post meta** / **ACF** — see `architecture.md` and `cursor-rules/acf-field-map.md`.

## Maintenance

When you change REST routes or add a new `install-*-tables.php` group:

1. Update the matching OpenAPI file and/or `database-overview.md`.
2. Mention the behavioural change in `architecture.md` if it is user- or operator-visible.
