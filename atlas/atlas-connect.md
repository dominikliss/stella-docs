# Atlas Connect

**Atlas Connect** is the WordPress plugin installed on each site Atlas manages. It lets an Atlas operator open a **one-time WP admin login URL** without sharing the site password.

Uptime pings do **not** use the plugin — those are a plain `HEAD https://{domain}` from Atlas. See [`monitoring.md`](monitoring.md).

System context: [`../integration/system-overview.md`](../integration/system-overview.md).

---

## Why the call is server-side

Atlas calls the plugin from **Laravel**, not from the browser:

1. The Atlas host IP is the address the site must allowlist (plugin returns **403** otherwise).
2. The shared secret never goes to the operator’s browser after it is saved.

UI: WP Sites drawer → **03. Atlas Connect**. Operator pastes the secret from **WP Admin → Settings → Atlas Connect → Connection**.

---

## What Atlas stores

On `wp_sites`:

| Column | Notes |
|--------|-------|
| `secret_key` | Encrypted (`encrypted` cast). Hidden from JSON. API exposes `has_secret_key` only. Empty PATCH does not overwrite. |
| `wp_user` | Optional WP username to log in as. Blank → plugin default. |

---

## Login contract (Atlas-facing)

Atlas `POST /api/wp-sites/{id}/login` →

```
POST https://{domain}/wp-json/dl-connect/v1/generate-token
Content-Type: application/json

{
  "secret": "<stored secret_key>",
  "user": "<wp_user>"          // omitted when empty
}
```

Expected success body: `{ "url": "https://…" }`. Atlas returns that URL; the SPA opens it in a new tab.

| Site HTTP | Atlas HTTP | Meaning |
|-----------|------------|---------|
| 401 | 401 | Invalid secret, or site requires HTTPS |
| 403 | 403 | Atlas host IP is not allowlisted on the site |
| 404 | 404 | Plugin route or WP user missing |
| other non-2xx | 502 | Pass through plugin `message` / `error` when present |
| 2xx without `url` | 502 | Plugin did not return a login URL |
| transport error | 502 | Site unreachable |

No `secret_key` on the row → Atlas **422** (does not call the site).

---

## Operator setup

1. Install Atlas Connect on the WordPress site.
2. In WP Admin → Settings → Atlas Connect → Connection: copy the secret, allowlist the **Atlas server** IP (not the operator laptop).
3. In Atlas → WP Sites: paste secret + optional WP username.

---

## What the plugin is not

- Not an uptime or backup agent. Columns like `last_backup_at`, `ssl_expires_at`, `plugin_updates_count` exist on `wp_sites` for a future push/pull; the current Atlas job only writes uptime.
- Not related to ddashboard `dls/v1` auth. Different secret, different host.
- Not Stella. The generate-token request goes **Atlas → client site** only.
