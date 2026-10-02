# Atlas Connect — login

Atlas calls the plugin from **Laravel**, not from the browser, so the Atlas host IP is what the site allowlists and the shared secret never goes to the operator’s browser after it is saved.

UI today: WP Sites drawer → **03. Atlas Connect**.

Full contract index: [`README.md`](README.md).

---

## Flow

```
Operator → Atlas POST /api/wp-sites/{id}/login
Atlas    → POST https://{domain}/wp-json/atlas-connect/v1/generate-token
         ← { "url": "https://client.com/?atlas_autologin=<token>" }
Atlas    → returns url; SPA opens it in a new tab
Site     → sets WP auth cookie, redirects to /wp-admin/ (or ?redirect=)
```

---

## Plugin request

```http
POST https://{domain}/wp-json/atlas-connect/v1/generate-token
Content-Type: application/json

{
  "secret": "<stored secret_key>",
  "user":   "<wp_user>",
  "ttl":    90
}
```

| Field | Required | Notes |
|---|---|---|
| `secret` | yes | JSON body, not query string |
| `user` | **yes** | WP login. Missing → `400` `{ "error": "Missing user param" }` |
| `ttl` | no | Seconds, clamped to 30–300, default 90 |

Success `200`:

```json
{ "url": "https://client.com/?atlas_autologin=AbCdEfGh1234567890123456789012" }
```

The token is single-use: deleted **before** `wp_set_auth_cookie()`. Do not store or log `url`. Optional `?redirect=` on the login URL is `sanitize_url`’d.

---

## Atlas HTTP mapping

No `secret_key` on the row → Atlas **422** (do not call the site).

| Site HTTP | Atlas HTTP | Meaning |
|-----------|------------|---------|
| 401 | 401 | Auth failed: bad secret, site requires HTTPS, **or Atlas IP not allowlisted** |
| 400 | 400 | Missing `user` |
| 404 | 404 | Plugin route missing, or WP user does not exist |
| other non-2xx | 502 | Pass through plugin `error` / `message` |
| 2xx without `url` | 502 | Plugin did not return a login URL |
| transport error | 502 | Site unreachable |

The old `dl-connect/v1` path is gone. Call **`atlas-connect/v1`**. Sites that already ran `dl-connect` are migrated; the REST namespace is not.

---

## Operator setup

1. Install Atlas Connect on the WordPress site.
2. WP Admin → Settings → Atlas Connect → Connection: copy the secret; allowlist the **Atlas server** IP.
3. Atlas → WP Sites: paste secret + **WP username** (required).
