# Atlas Connect — login

Atlas calls the plugin from **Laravel**, not from the browser. The Atlas host IP is what the site allowlists for the API, and the shared secret never reaches the operator's browser after it is saved.

The login link itself is opened in the operator's browser and is locked to the **team office IP**.

UI today: WP Sites drawer → **03. Atlas Connect**.

Full contract index, auth, and secret lifecycle: [`README.md`](README.md). Backups: [`backups.md`](backups.md). Infection scan: [`security-scan.md`](security-scan.md).

---

## Flow

```
Operator (office, 194.126.177.181)
         → Atlas POST /api/wp-sites/{id}/login
Atlas    (2.31.42.13 prod / 95.217.144.93 dev)
         → POST https://{domain}/wp-json/atlas-connect/v1/generate-token
         ← { "url": "https://client.com/?atlas_autologin=<token>" }
Atlas    → returns url; SPA opens it in a new tab
Browser  (must be 194.126.177.181)
         → GET https://client.com/?atlas_autologin=<token>
Site     → sets WP auth cookie, fires wp_login, redirects to /wp-admin/ (or ?redirect=)
```

Two different IP gates:

| Step | Caller | Must come from | Config key |
|---|---|---|---|
| 1. `generate-token` | Atlas server | `2.31.42.13` or `95.217.144.93` | `config.php` → `ip_whitelist` |
| 2. Open the link | Operator's browser | `194.126.177.181` (team office static IP) | `config.php` → `autologin_ip_whitelist` |

Both use the same client-IP logic (`REMOTE_ADDR`, or the rightmost `X-Forwarded-For` entry when `REMOTE_ADDR` is a trusted proxy). See [`README.md`](README.md#how-the-client-ip-is-determined).

---

## Step 1 — plugin request

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
| `secret` | yes | **JSON body only.** `?secret=` in the URL is ignored → `401`. Must be a string |
| `user` | no | WP **login name** (not email, not display name). Must be an **administrator** (or super admin on multisite). If omitted, the plugin uses the site's “Auto-login Username” setting (`atlas_autologin_user`). If both are empty → `400`. Atlas should always send it |
| `ttl` | no | Seconds the link stays valid. Integer, clamped to **30–300**, default **90** |

Success `200`:

```json
{ "url": "https://client.com/?atlas_autologin=AbCdEfGh1234567890123456789012" }
```

- The URL is `home_url('/')` + `?atlas_autologin=` + a 32-char `[A-Za-z0-9]` token.
- The token is stored as transient `atlas_autologin_{token}` → username, expiring after `ttl`.
- Do not store or log `url`. It is a bearer credential until used or expired.

### Step 1 errors

| HTTP | Body | When |
|---|---|---|
| `401` | `{ "code": "rest_forbidden", "message": "Sorry, you are not allowed to do that.", "data": { "status": 401 } }` | Auth failed: not HTTPS (when required), Atlas IP not in `ip_whitelist`, secret missing / wrong / in query string / not a string, or the site has no secret yet. The plugin does not say which |
| `400` | `{ "error": "Missing user param" }` | No `user` in the body and no Auto-login Username set on the site |
| `403` | `{ "error": "Only administrators can be logged in via Atlas." }` | The user exists but is not an administrator (editors, shop managers, custom roles with `manage_options` but no `administrator` role — all rejected) |
| `404` | `{ "error": "User not found." }` | No WP user with that login |
| `404` | `{ "code": "rest_no_route", … }` | Plugin not active, wrong namespace (`dl-connect/v1`), host strips the namespace, or the request was sent as `GET` |

---

## Step 2 — opening the link

The SPA opens `url` in a new tab. On WordPress `init` the plugin:

1. Reads `atlas_autologin` from the query string.
2. **Checks the browser's IP** against `autologin_ip_whitelist` (`194.126.177.181`). Not on the list → `403` HTML page “Login link expired or invalid.” **The token is not consumed** — the same link still works from the office until it expires.
3. Looks up the transient. Missing (expired or already used) → `403` HTML page “Login link expired or invalid.”
4. **Deletes the token** (single-use) **before** logging in.
5. Looks up the user again. Deleted in the meantime → `404` HTML page “User not found.”
6. `wp_set_auth_cookie( $user_id, false )` — a session cookie (no “remember me”).
7. `wp_set_current_user()` + `do_action( 'wp_login', $login, $user )`, so activity-log / security plugins (WP Activity Log, Wordfence, Simple History, …) record the login like a normal one.
8. Redirects: `?redirect=` if present and same-site (`wp_safe_redirect()`); otherwise, or for other hosts, WP admin (`admin_url()`).

The wrong-IP page and the expired page are deliberately identical, so an outsider holding a leaked link learns nothing.

### Optional redirect

Append `&redirect=` (URL-encoded) to the returned `url` to land somewhere other than the dashboard:

```
https://client.com/?atlas_autologin=AbCd…&redirect=https%3A%2F%2Fclient.com%2Fwp-admin%2Fplugins.php
```

Same-site only. Other hosts fall back to `admin_url()`.

### What the operator sees off-network

Home office, VPN, mobile hotspot, café: the tab shows “Login link expired or invalid.” This is expected. Either connect through the office network within the TTL, or ask for the IP to be added to `autologin_ip_whitelist` (plugin release — it is code in `config.php`, not a WP setting).

---

## Atlas HTTP mapping

No `secret_key` on the row → Atlas **422** (do not call the site).

| Site HTTP | Atlas HTTP | Atlas message |
|-----------|------------|---------------|
| 401 | 401 | “Atlas Connect rejected the request — check the secret, HTTPS, and that the Atlas IP is allowlisted.” (Plugin IP failures are `401`, **not** `403`) |
| 400 | 400 | “No WP user given and the site has no Auto-login Username.” |
| 403 | 403 | “WP user `{user}` is not an administrator on this site.” |
| 404 with `error` | 404 | “WP user `{user}` does not exist.” |
| 404 with `code: rest_no_route` | 502 | “Atlas Connect is not active on the site (or the REST namespace is blocked).” |
| other non-2xx | 502 | Pass through `error ?? message` |
| 2xx without `url` | 502 | Plugin did not return a login URL |
| transport error | 502 | Site unreachable. Include the transport detail, e.g. `Could not reach the site (cURL error 7: …). Is it online?` |

Parse errors as `data.error ?? data.message`. `401` / `rest_no_route` bodies have no `error` key.

If Docker DNS maps the site hostname to a private container IP (no TLS on :443), Atlas pins `CURLOPT_RESOLVE` to the public A record from Cloudflare DNS before POSTing.

The old `dl-connect/v1` path is gone. Call **`atlas-connect/v1`**. Sites that already ran `dl-connect` are migrated; the REST namespace is not.

---

## Suggested Atlas call (TypeScript)

```ts
type LoginResult = { url: string };

export async function getLoginUrl(site: { url: string; secretKey: string; wpUser: string }): Promise<string> {
  const res = await fetch(`${site.url.replace(/\/$/, '')}/wp-json/atlas-connect/v1/generate-token`, {
    method:  'POST',
    headers: { 'Content-Type': 'application/json' },
    body:    JSON.stringify({ secret: site.secretKey, user: site.wpUser, ttl: 90 }),
    signal:  AbortSignal.timeout(15_000),
  });

  const data = await res.json().catch(() => ({} as Record<string, unknown>));
  if (!res.ok) {
    const msg = (data as any).error ?? (data as any).message ?? 'unknown error';
    throw new Error(`Login failed (${res.status}): ${msg}`);
  }
  if (typeof (data as LoginResult).url !== 'string') {
    throw new Error('Plugin did not return a login URL');
  }
  return (data as LoginResult).url; // open in a new tab; never log or store
}
```

---

## Operator setup

1. Install Atlas Connect on the WordPress site.
2. WP Admin → Settings → Atlas Connect → Connection → **Generate**, then **Copy** — the key is shown once.
3. Pick **Auto-login Username** (administrators only) and **Save Connection Settings**.
4. Atlas → WP Sites: paste the secret + **WP username** (an administrator).
5. Open the login from the office network (`194.126.177.181`).

The Atlas server IP is already in the shipped `ip_whitelist`. The operator does not allowlist anything per site.
