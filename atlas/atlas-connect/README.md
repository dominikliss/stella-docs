# Atlas Connect

**Atlas Connect** is the WordPress plugin on each site Atlas manages. Atlas is always the caller. The site never phones home.

It does two jobs:

1. **One-time WP admin login** — [`login.md`](login.md)
2. **Encrypted chunked backups** to S3-compatible storage — [`backups.md`](backups.md)

Uptime pings do **not** use the plugin (`HEAD https://{domain}`). See [`../monitoring.md`](../monitoring.md).

| | |
|---|---|
| Plugin REST | `https://{site_url}/wp-json/atlas-connect/v1` |
| Plugin admin | Settings → Atlas Connect |
| Plugin repo | `git@github.com:foxcraftdigital/atlas-connect.git` |

System context: [`../../integration/system-overview.md`](../../integration/system-overview.md).

---

## What Atlas must store

### Per WordPress site (`wp_sites`)

| Field | Source | Notes |
|---|---|---|
| Domain / `site_url` | Site home URL | No trailing path. Example: `https://client.com` |
| `secret_key` | WP Admin → Settings → Atlas Connect → Connection → Copy | Encrypt at rest. Hide from JSON (`has_secret_key` only). Empty PATCH must not overwrite. Regenerating the secret in WP invalidates Atlas until updated |
| `wp_user` | WP login on that site | **Required** on every plugin `generate-token` call. The plugin does not fall back to the WP “Auto-login Username” setting |

Atlas’s **outbound server IP** must be in that site’s `config.php` → `ip_whitelist` (code on the site, not a WP setting). Allowlist the Atlas host, not the operator laptop.

### Once, for the fleet (Atlas vault)

| Field | Role |
|---|---|
| X25519 **key pair** | Public half (64 hex chars) goes in every site’s `config.php` → `backup.public_key`. Private half never leaves Atlas |
| S3 `endpoint`, `bucket`, `region` | e.g. `https://fsn1.your-objectstorage.com`, `fsn1` |
| S3 `access_key` + `secret_key` | Sent on every backup start / chunk / cancel. **Not** stored on the WordPress site |

Encrypt site secrets, S3 keys, and the sodium private key at rest. Never log them. Never log a generated login URL.

One X25519 pair for the whole fleet is fine.

---

## Security rules

1. **Site never calls Atlas.** No callbacks, license checks, or “fetch my key”.
2. **Only Atlas starts backups.** WP admin has history only — no Start button, no S3 fields.
3. **`secret` and S3 keys go in the JSON body** of POSTs. Never as query parameters (access logs).
4. `GET /backup/status/{job_id}` is GET-only, so `?secret=` will appear in the site’s access logs. Do **not** put S3 keys on that URL.
5. Send S3 credentials on **start, chunk, and cancel**. The plugin uses them for that request only and does not persist them.
6. Prefer a **per-site prefix-scoped** S3 key (`backups/{site_id}/*`) if the provider allows it. Hetzner keys are long-lived; “short-lived” only means the site does not keep them.
7. Drive `/backup/chunk` **sequentially**. Do not parallelize the same `job_id`.

---

## Authentication (every plugin route)

`atlas_secret_authentication()`. Failure is always WordPress REST **`401`**. The plugin does not say whether SSL, IP, or secret failed.

All of these fail the same way:

- HTTP while the site requires SSL (`atlas_require_ssl`, default on for production)
- `REMOTE_ADDR` not in `ip_whitelist`
- `secret` missing or not equal to `atlas_secret_key` (`hash_equals`)

Send `secret` in the JSON body (except GET status).

Atlas today maps plugin `403` → “IP not allowlisted”. The current plugin returns **`401` for IP as well**. Update that mapping when implementing against this contract.

---

## Site onboarding

On the WordPress site

1. Install / activate `atlas-connect`.
2. Copy the **Secret Key** into Atlas.
3. Put Atlas’s outbound IP in `config.php` → `ip_whitelist`.
4. Put Atlas’s X25519 **public** key hex in `config.php` → `backup.public_key`.
5. PHP `sodium` must be enabled (backup start → `500` otherwise).
6. REST allowlists must keep `atlas-connect` (some hosts strip unknown namespaces → `404 rest_no_route`).

In Atlas

1. Keep the X25519 pair and S3 keys in the vault — never in WP.
2. Store `secret_key` + `wp_user` per site.
3. Login: [`login.md`](login.md). Backups: [`backups.md`](backups.md).

No S3 setup is done in WordPress.

---

## Atlas Laravel work still to do

The plugin contract is live. Atlas still needs:

| Area | Today | Implement |
|---|---|---|
| Login | `POST /api/wp-sites/{id}/login` → plugin `generate-token` | Switch path from `dl-connect/v1` to **`atlas-connect/v1`**. Always send `user`. Treat plugin `401` as auth (incl. IP) |
| Backup | Not implemented (`last_backup_at` is unused) | Driver in [`backups.md`](backups.md): start → loop chunk → read `manifest.json` from the bucket. Vault holds S3 + private key |
| WP Sites UI | Secret + optional user | Require `wp_user`. Add backup start / progress / history |

---

## What the plugin is not

- Not an uptime agent.
- Not ddashboard `dls/v1` auth.
- Not Stella. Traffic is **Atlas → client site** (and site → S3 during a backup). Atlas later reads S3 with its own keys.
