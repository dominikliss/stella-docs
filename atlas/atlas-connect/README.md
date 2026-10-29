# Atlas Connect

**Atlas Connect** is the WordPress plugin on each site Atlas manages. Atlas is always the caller. The site never phones home.

It does four jobs:

1. **One-time WP admin login** — [`login.md`](login.md)
2. **Encrypted chunked backups** to S3-compatible storage — [`backups.md`](backups.md)
3. **Infection scan** (file integrity + persistence) — [`security-scan.md`](security-scan.md)
4. **Version reporting and Atlas-pushed plugin updates** (signed zip in the request) — [`updates.md`](updates.md)

Uptime pings do **not** use the plugin (`HEAD https://{domain}`). See [`../monitoring.md`](../monitoring.md).

| | |
|---|---|
| Plugin REST | `https://{site_url}/wp-json/atlas-connect/v1` — or `/{rest_prefix}/…` when the site renames its REST API (`rest_url_prefix` filter); set per site in Atlas (Edit site → Atlas Connect → REST API Prefix, column `wp_sites.rest_prefix`, blank = `wp-json`) |
| Plugin admin | Settings → Atlas Connect |
| Plugin repo | `git@github.com:foxcraftdigital/atlas-connect.git` |

System context: [`../../integration/system-overview.md`](../../integration/system-overview.md).

**What changed in the security hardening (2026-10-23) and what Atlas must change:** [Atlas work still to do](#atlas-work-still-to-do). Read that section first if you are updating Atlas.

---

## Route overview

Every route is **`POST`** with a **JSON body**. There are no `GET` routes any more.

| Route | Purpose | Body (besides `secret`) | Doc |
|---|---|---|---|
| `POST /generate-token` | One-time login URL | `user?`, `ttl?` | [`login.md`](login.md) |
| `POST /backup/start` | Start a backup job | `backup_type?`, `s3` | [`backups.md`](backups.md) |
| `POST /backup/chunk` | Do the next slice of work | `job_id`, `s3` | [`backups.md`](backups.md) |
| `POST /backup/status/{job_id}` | Read progress, no work | — | [`backups.md`](backups.md) |
| `POST /backup/cancel` | Cancel a running backup | `job_id`, `s3` | [`backups.md`](backups.md) |
| `POST /security/scan/start` | Start an infection scan | `allowlist?` | [`security-scan.md`](security-scan.md) |
| `POST /security/scan/chunk` | Do the next slice of work | `job_id` | [`security-scan.md`](security-scan.md) |
| `POST /security/scan/status/{job_id}` | Read progress / report, no work | — | [`security-scan.md`](security-scan.md) |
| `POST /info` | Plugin / WP / PHP versions, update readiness, running jobs. No work | — | [`updates.md`](updates.md#post-info) |
| `POST /plugin/update` | Install a signed plugin zip sent by Atlas | `version`, `sha256`, `signature`, `package` | [`updates.md`](updates.md#post-pluginupdate) |

A `GET` on any of these returns WordPress's `404`:

```json
{ "code": "rest_no_route", "message": "No route was found matching the URL and request method.", "data": { "status": 404 } }
```

---

## What Atlas must store

### Per WordPress site (`wp_sites`)

| Field | Source | Notes |
|---|---|---|
| Domain / `site_url` | Site home URL | No trailing path. Example: `https://client.com` |
| `secret_key` | WP Admin → Settings → Atlas Connect → Connection → **Generate** / **Regenerate**, then **Copy** | 32 chars `[A-Za-z0-9]`. **Shown once.** The site stores only a SHA-256 of it, so it cannot be read back from WordPress later. Encrypt at rest in Atlas. Hide from JSON (`has_secret_key` only). Empty PATCH must not overwrite. Regenerating in WP invalidates the old key immediately |
| `wp_user` | WP login on that site | Must be an **administrator** (or super admin on multisite). Optional: if Atlas omits `user`, the plugin uses the site's “Auto-login Username” setting. Atlas should still send it explicitly |

Optional but useful: store the **last 4 characters** of `secret_key` (or derive them when displaying). WP admin shows the key as `****************************abcd`, so an operator can check which key a site expects without either side revealing it.

### Once, for the fleet (Atlas vault)

| Field | Role |
|---|---|
| X25519 **key pair** | Public half (64 hex chars) goes in every site’s `config.php` → `backup.public_key`. Atlas stores only the public half. The private half stays with the operator (password manager / offline) and is pasted into the browser when decrypting; Atlas never receives it |
| S3 `endpoint` | **Must** be `https://fsn1.your-objectstorage.com`, `https://nbg1.your-objectstorage.com`, or `https://hel1.your-objectstorage.com`. Anything else → `400` on every backup call |
| S3 `bucket` | **Must** be `atlas-wp-sites-backups`. Anything else → `400` on every backup call |
| S3 `region` | e.g. `fsn1` (default if omitted) |
| S3 `access_key` + `secret_key` | Sent on every backup start / chunk / cancel (not status). **Not** stored on the WordPress site |

Encrypt site secrets and S3 keys at rest. Never log them. Never log a generated login URL.

One X25519 pair for the whole fleet is fine.

---

## Security rules

1. **Site never calls Atlas.** No callbacks, license checks, or “fetch my key”.
2. **Only Atlas starts backups and scans.** WP admin has backup history only — no Start button, no S3 fields, no scan results.
3. **Every route is `POST`.** `secret`, S3 keys, and the scan allowlist go in the **JSON body**. The plugin ignores `?secret=` in the query string — a request that only has it there fails with `401`.
4. **Only Atlas's IPs can call the REST API** (see [IP allowlist](#ip-allowlist)). Only the **team office IP** can open a login link (see [`login.md`](login.md)).
5. Send S3 credentials on **start, chunk, and cancel**. The plugin uses them for that request only and does not persist them.
6. **Backups can only go to bucket `atlas-wp-sites-backups` on Hetzner Object Storage over HTTPS** (see [`backups.md`](backups.md#s3-destination-is-pinned)).
7. Send the scan **allowlist** on **start** only. Path + SHA-256. The site does not persist it.
8. Prefer a **per-site prefix-scoped** S3 key (`backups/{hostname}/*`) if the provider allows it. Hetzner keys are long-lived; “short-lived” only means the site does not keep them.
9. Drive `/backup/chunk` and `/security/scan/chunk` **sequentially**. Do not parallelize the same `job_id`.
10. Login links are only issued for **administrators**.

---

## Authentication (every plugin route)

`atlas_secret_authentication()` runs as the `permission_callback` on all 10 routes. It checks, in this order, and stops at the first failure:

| # | Check | Passes when |
|---|---|---|
| 1 | **SSL** | `is_ssl()` is true, or SSL is not required on that site (see below) |
| 2 | **Client IP** | The client IP (see [How the client IP is determined](#how-the-client-ip-is-determined)) is in `config.php` → `ip_whitelist` |
| 3 | **Secret** | `secret` from the **JSON body** (or form body), is a **string**, non-empty, and `hash_equals( stored_sha256, sha256( secret ) )` |

Any failure returns WordPress's standard REST error with HTTP **`401`**:

```json
{ "code": "rest_forbidden", "message": "Sorry, you are not allowed to do that.", "data": { "status": 401 } }
```

Note the shape: **`code` / `message`, not `error`**. Route-level errors (after auth passed) use `{ "error": "…" }`; `/plugin/update` adds a machine-readable `"code"` next to `error`. Atlas's error parsing must handle both, e.g. `data.error ?? data.message`.

The plugin never says which check failed. Atlas cannot tell SSL, IP, and secret apart, so the UI message should name all three: “Atlas Connect rejected the request — check the secret, that the site is on HTTPS, and that the Atlas IP is allowlisted.”

**Temporary:** while a 401 is being diagnosed, every auth attempt is appended to `wp-content/atlas-connect-debug/auth.log` on the site (result, SSL/proxy headers, client IP, last 4 characters of the provided secret, secret hint). The plaintext secret is never logged. This logging will be removed again.

Things that now fail with `401` that used to work:

- `secret` only in the query string (`?secret=…`).
- `secret` sent as a non-string (number, array, object).
- A site that was never given a secret (fresh install, nobody clicked **Generate**).

### SSL requirement

- Setting: WP Admin → Atlas Connect → Connection → **Require SSL** (`atlas_require_ssl`).
- If the setting was never saved, the default is **on**, unless `wp_get_environment_type()` is `local` or `development` (set via `WP_ENVIRONMENT_TYPE` in `wp-config.php`).
- WordPress treats an unset environment type as `production`, so **dev sites on plain HTTP now require HTTPS** unless they set `WP_ENVIRONMENT_TYPE` or untick Require SSL.
- The old rule (SSL off when the `Host` header contained `dev.`, `local.`, or `.test`) is gone — the client controls that header.
- Behind a TLS-terminating proxy, `is_ssl()` needs `$_SERVER['HTTPS'] = 'on'` set from `X-Forwarded-Proto` in `wp-config.php` (see [`../../stella-server/wordpress-staging.md`](../../stella-server/wordpress-staging.md)).

### IP allowlist

`config.php` → `ip_whitelist` (code in the plugin repo, not a WP setting — changing it means a plugin release). `tests/release-test.php` fails CI if `2.31.42.13` or `95.217.144.93` is missing — a release without them locks Atlas out, and Atlas could not push the fix:

| IP | Who |
|---|---|
| `2.31.42.13` | Atlas production (`atlas.foxcraft.digital`, Hetzner CX33 `atlas`) |
| `95.217.144.93` | Atlas dev instance on Stella (`atlas.dev`). Note: anything else running on Stella shares this IP |
| `172.20.0.1` | Docker `edge` gateway on Stella. Matches only when a request starts on Stella itself (host or a container) and loops back through Caddy to a site hosted on Stella — e.g. Atlas dev calling a Stella-hosted site. Outside traffic never matches it (see below) |
`23.88.90.12` (managed host) was **removed**. Allowlist the Atlas host, never an operator laptop.

### How the client IP is determined

`atlas_client_ip()`:

1. If `REMOTE_ADDR` is **not** in `config.php` → `trusted_proxies`, the client IP is `REMOTE_ADDR`.
2. If `REMOTE_ADDR` **is** a trusted proxy and `X-Forwarded-For` is missing or empty, the client IP is `REMOTE_ADDR`. Either the request really came from the proxy host, or Apache's `mod_remoteip` (enabled in the official `wordpress` image) already moved the client IP into `REMOTE_ADDR` and dropped the header.
3. Otherwise the client IP is the **rightmost** entry of `X-Forwarded-For` (the one the proxy itself appended). Entries to the left are client-supplied and ignored. If that entry is not a valid IP, the client IP is empty → `401`.

`trusted_proxies` today: `172.20.0.1` (Docker `edge` gateway on Stella — Caddy → WordPress).

`172.20.0.1` is in **both** lists. Before 1.0.7 the plugin compared `REMOTE_ADDR` directly, so having Caddy's address in `ip_whitelist` let **every** request routed through Caddy pass the IP check. Now the check uses the IP Caddy reports in `X-Forwarded-For`: an outside caller shows up as its public IP and must be `2.31.42.13` / `95.217.144.93`; only traffic that reaches Caddy from Stella's own Docker network shows up as `172.20.0.1` and matches that entry. The rightmost `X-Forwarded-For` entry is always the address Caddy itself received the connection from, so an outsider cannot fake it.

Requirements for this to work:

- Caddy's `reverse_proxy` must send `X-Forwarded-For` (it does by default). If it doesn't, every request — Atlas's included — gets `401`.
- Atlas dev on Stella calling a site that is also on Stella hairpins through Caddy. If the source Caddy sees is the `edge` gateway `172.20.0.1`, it is allowlisted. If it is some other Docker address (a container on a different network, e.g. `172.18.x.x`), it is **not** → `401`. Test `generate-token` from Atlas dev once after deploying and add that address if needed.
- If another proxy (e.g. Cloudflare) is ever put in front of Caddy, the rightmost entry becomes that proxy's address and Atlas would `401`. The plugin would then need to trust that hop too.
- IPv6: neither Atlas IP has an IPv6 entry. If Atlas ever connects over IPv6 it fails closed (`401`). Force IPv4 in the Atlas HTTP client if that happens.

---

## Secret key lifecycle

| Event | What happens on the site | What Atlas does |
|---|---|---|
| Plugin activated (fresh install) | **No secret exists.** Every REST call → `401` | Nothing until the operator generates one |
| Operator clicks **Generate** | New 32-char secret; the plugin stores `sha256(secret)` (`atlas_secret_hash`) + last 4 chars (`atlas_secret_hint`). The full key is shown in the page **once**, with a Copy button and a warning | Operator pastes it into Atlas → WP Sites → site |
| Operator leaves / reloads the page | Key is gone from WP for good. WP shows `****…abcd` only | — |
| Operator clicks **Regenerate** | Old key stops working **immediately**. New key shown once | Operator updates Atlas. Until then all calls `401` |
| Existing site upgrades to this plugin version | On first page load the old plaintext `atlas_secret_key` is hashed into `atlas_secret_hash` and **deleted**. The key Atlas already has keeps working | Nothing — but nobody can read the key from WP any more. If Atlas lost it, regenerate |

There is no way to fetch the secret from the site. Atlas is the only place it exists in plaintext after the copy.

---

## Site onboarding

On the WordPress site

1. Install / activate `atlas-connect`. On **WP Sites**, **Atlas Connect ▾ → Download zip** always clones the current `main` of `foxcraftdigital/atlas-connect` (same SSH key and excludes as deployments: `.git`, `.cursor`, `docs`) and saves a zip you can upload in WP Admin → Plugins. Nothing is cached — every click rebuilds.
2. Settings → Atlas Connect → Connection → **Generate**. Copy the key **now** into Atlas.
3. Pick the **Auto-login Username** (administrators only in the dropdown) and **Save**. Used when Atlas omits `user`.
4. Make sure Atlas's IP is in `config.php` → `ip_whitelist` (already true for `2.31.42.13` and `95.217.144.93` in the shipped config).
5. `config.php` → `backup.public_key` holds Atlas's X25519 **public** key hex (shipped in the plugin). `update.public_key` holds the separate Ed25519 key that signs plugin updates ([`updates.md`](updates.md#signing-key)).
6. PHP `sodium` must be enabled (backup start → `500` otherwise).
7. REST allowlists must keep `atlas-connect` (some hosts strip unknown namespaces → `404 rest_no_route`).
8. Plain-HTTP dev site? Set `WP_ENVIRONMENT_TYPE` to `local` or `development`, or untick **Require SSL**.
9. Behind Caddy on Stella: make sure Caddy sends `X-Forwarded-For` and `X-Forwarded-Proto`.

In Atlas

1. Keep the X25519 pair and S3 keys in the vault — never in WP.
2. Store `secret_key` + `wp_user` per site.
3. Login: [`login.md`](login.md). Backups: [`backups.md`](backups.md). Scans: [`security-scan.md`](security-scan.md).

No S3 setup is done in WordPress.

---

## Atlas work still to do

### Required by the 2026-10-23 plugin hardening

These break against the new plugin until Atlas is updated.

**Atlas status (2026-10-23): done in the Atlas tree.**

- Rows 1–3: Atlas drives with `chunk` and does not poll the status routes for progress. It only calls `POST /backup/status/{job_id}` (`AtlasConnectClient::backupStatus()`) to recover from a site-side `409` on start (see [Backup job recovery](#backup-job-recovery)). Every plugin call is `POST` + JSON body (`AtlasConnectClient::post()`).
- Row 4: `AtlasConnectClient::decode()` reads `error ?? message` and keeps WP's `code` on `AtlasConnectException::$pluginCode`.
- Row 5: `BackupVault::assertAllowedDestination()` (`ALLOWED_BUCKET`, `ALLOWED_ENDPOINT_HOSTS`). Settings → Backups rejects other values (`422`, plus inline checks in the form). Start / chunk / cancel use `pluginS3CredentialsFor()` and fail in Atlas without calling the site. Atlas’s own bucket reads (explorer, delete, manifest) are not pinned, so older backups stay readable.
- Row 6: login maps `401` / `400` / `403` (not admin) / `404` (no such user) / `404 rest_no_route` → `502`, with the messages in [`login.md`](login.md#atlas-http-mapping). Scan chunks treat `403` as fatal.
- Rows 7–9: WP Sites form and login toast copy (Generate + shown once, administrator only, office network only).
- Row 10: `AtlasConnectClient` forces IPv4 (`CURLOPT_IPRESOLVE`). Which public IP the host uses is infrastructure, not code. Still test it once from prod and dev (see [`../../open-gaps.md`](../../open-gaps.md)).

| # | Area | Change in Atlas | Why |
|---|---|---|---|
| 1 | Backup status | `GET /backup/status/{job_id}?secret=…` → **`POST /backup/status/{job_id}`** with JSON `{ "secret": "…" }` | `GET` routes were removed (`404 rest_no_route`). Secret no longer accepted in the URL |
| 2 | Scan status | `GET /security/scan/status/{job_id}?secret=…` → **`POST /security/scan/status/{job_id}`** with JSON `{ "secret": "…" }` | Same |
| 3 | Every call | Never put `secret` in the query string. Always JSON body, `Content-Type: application/json` | Query-string secret is ignored → `401` |
| 4 | Error parsing | Read `error`, fall back to `message` (`data.error ?? data.message`). `401` bodies are WP's `{ code, message, data }` | Auth failures have no `error` key |
| 5 | S3 vault validation | In Atlas Settings, reject any bucket other than `atlas-wp-sites-backups` and any endpoint other than `https://{fsn1,nbg1,hel1}.your-objectstorage.com` | The site now refuses them with `400` on start, chunk, and cancel. Fail fast in Atlas instead |
| 6 | Login HTTP mapping | Add plugin **`403`** → Atlas `403` “WP user is not an administrator on this site”. Plugin IP failures are `401`, not `403` | New check in `generate-token` |
| 7 | Login UI | Tell the operator the link only works from the **office network** (`194.126.177.181`). A `403` page “Login link expired or invalid.” from another network is expected | Browser step is now IP-locked |
| 8 | Site onboarding UI | Copy: “In WP → Settings → Atlas Connect click **Generate**, copy the key — it is shown only once.” Remove any mention of copying an existing key later | Secret is hash-only on the site |
| 9 | `wp_user` field | Hint: “must be an administrator”. Keep sending it | Non-admins → `403` |
| 10 | Outbound IP | Make sure Atlas prod calls sites from `2.31.42.13` and Atlas dev from `95.217.144.93` (or via Caddy's `edge` gateway `172.20.0.1` for Stella-hosted sites) over **IPv4** | Only these are allowlisted |

### Backup job recovery

Atlas and the site each track a job's status. They can disagree when a chunk call fails.

- **Chunk fails** (timeout, `5xx`, unreachable): `WpSiteBackupDriver::drive()` calls `/backup/cancel` for that job, then marks the Atlas row `error`. If the cancel also fails, the site keeps the job `running` until its TTL (24h).
- **Start returns `409`** and Atlas has no `pending` / `running` row: Atlas checks its last 5 `error` / `cancelled` rows from the past 24h that have a `plugin_job_id`, asking `/backup/status/{job_id}` for each.
  - Atlas row `error`, site `running`: the row goes back to `running` with the site's progress and `RunWpSiteBackupJob` is queued again. The job keeps its original `backup_type`; the UI toast says "Resumed".
  - Atlas row `cancelled`, site `running`: Atlas cancels it on the site and retries the start once.
  - No match: `409` "The site reports a backup is already running that Atlas is not tracking. It clears once that job expires (up to 24h)."

### Still open from before

| Area | Today | Implement |
|---|---|---|
| Login | `POST /api/wp-sites/{id}/login` → plugin `generate-token` | Path is **`atlas-connect/v1`** (not `dl-connect/v1`). Send `user`. Map `401` (auth incl. IP), `403` (not admin), `404` (no such user) |
| Backup | Implemented: `WpSiteBackupDriver` + `RunWpSiteBackupJob`; Settings vault; WP Sites start / progress / history / explorer | Apply rows 1, 3–5 above. Keep in line with [`backups.md`](backups.md) |
| Infection scan | Implemented: `WpSiteScanDriver` + `RunWpSiteScanJob` (`scans` queue), twice-daily `wp-sites:run-daily-scans`, inventory + report + verdict + baseline + allowlist in DB, `wp_site.security_scan` alerts. See [`security-scan.md`](security-scan.md#atlas-implementation) | Apply rows 2–4 above. Keep in line with [`security-scan.md`](security-scan.md) |
| WP Sites UI | Secret + `wp_user`; backup panel; security scan panel (start, progress, history, report modal with allowlist + trust baseline); scan badge in the site list | Rows 7–9 above |

---

## Plugin updates

Atlas pushes plugin updates over the plugin's own REST API. Sites never fetch anything from GitHub and Atlas needs no SSH access to them.

Plugin-side contract (`/info` fields, `/plugin/update` request, check order, error codes, downgrade policy, first manual install): [`updates.md`](updates.md).

| Step | What happens |
|---|---|
| Release | Tag the plugin repo `vX.Y.Z` (or `X.Y.Z`). The main plugin file's `Version:` header must equal the tag number, or the build fails |
| Version check | `POST /info` (plugin) → `wp_sites.atlas_connect_version`, `atlas_connect_info` (capabilities), `atlas_connect_error`, `atlas_connect_checked_at`. Daily `wp-sites:check-atlas-connect`, the sync button on the site's Atlas Connect tile, or **Atlas Connect ▾ → Sync all** (latest tag without the 1 h cache, then every site with a secret key, 3 at a time). The WP Sites table's **Atlas Connect** column shows each site's version: up to date, outdated, check failed, or updating. A `404 rest_no_route` means the plugin predates `/info`: upload the zip once manually |
| Queue | WP Sites tile → update button (one site), or **Atlas Connect ▾ → Update all** (every outdated site whose `/info` says `update.supported`). One row per site in `wp_site_plugin_updates` (`queued` → `running` → `complete` / `error` / `cancelled`) |
| Run | `AtlasConnectUpdater::drain()` runs one update at a time (cache lock), from the `plugin-updates` queue or the every-minute scheduler fallback. Per site: no backup / scan running → `/info` capability check → build the tag's zip once (`storage/app/atlas-connect-releases/{version}.zip`) → sign → `POST /plugin/update` with base64 zip → `/info` must report the new version |
| Rollout | Staging sites first (or the chosen canary). The first failure cancels the rest of the batch |
| Timeout | A push that times out or returns a bare 500 is still `complete` if the follow-up `/info` reports the new version |

Signing: Ed25519. Atlas holds `ATLAS_CONNECT_UPDATE_SIGNING_KEY` (`php artisan atlas-connect:signing-key --generate`); the public half ships in the plugin `config.php` → `update.public_key`. The signed message is `atlas-connect-update\n{version}\n{sha256}` (LF, no trailing newline, sha256 of the zip bytes in lowercase hex). Without the key, Atlas hides the update buttons and the API returns `422`. Changing the key needs a plugin release with the new public key on every site first.

`/plugin/update` error codes Atlas shows as-is (`error (code)`): `update_disabled`, `filesystem_not_writable`, `update_in_progress`, `job_running`, `invalid_version`, `downgrade_refused`, `invalid_request`, `package_too_large`, `hash_mismatch`, `bad_signature`, `invalid_package`, `install_failed`.

---

## Plugin internals (not required to implement Atlas)

| Thing | Where |
|---|---|
| Auth | `authentication.php` → `atlas_secret_authentication()`, `atlas_client_ip()`, `atlas_store_secret()` |
| IP lists, trusted proxies, S3 pins, scratch dir | `config.php` (`ip_whitelist`, `trusted_proxies`, `autologin_ip_whitelist`, `backup.s3_bucket`, `backup.s3_endpoint_hosts`, `scratch_dir`) |
| Secret storage | Options `atlas_secret_hash` (sha256 hex) + `atlas_secret_hint` (last 4 chars). No plaintext option. Old `atlas_secret_key` is migrated and deleted |
| Scratch files | `atlas_scratch_dir()` in `lib/functions.php`. Private `0700` dir under `sys_get_temp_dir()/atlas-connect-{md5(ABSPATH)[0:12]}/`. Fallback when that is not writable (e.g. `open_basedir`): `uploads/atlas-{backup,scan}-tmp-{24 random chars}/` with a deny-all `.htaccess`. The old fixed `uploads/atlas-backup-tmp/` and `uploads/atlas-scan-tmp/` are deleted on upgrade |
| Upgrade hook | `plugins_loaded`, when `atlas_db_version` ≠ `1.2`: hash the secret, delete legacy scratch dirs, run `dbDelta` |
| Version | Only the `Version:` header in `atlas-connect.php` (= release tag). `ATLAS_CONNECT_VERSION` is read from it at load |
| Updates | `modules/update/` → `UpdateService` (`/info`, `/plugin/update`), lock option `atlas_update_lock` |
| Direct file access | Every PHP file starts with `defined( 'ABSPATH' ) \|\| exit;`. Files under `tests/` exit unless run from the CLI |
| Tests | `php tests/run.php` (standalone, WordPress stubbed). GitHub Actions runs it on every push and tag |

---

## What the plugin is not

- Not an uptime agent.
- Not ddashboard `dls/v1` auth.
- Not Stella. Traffic is **Atlas → client site** (and site → S3 during a backup). Atlas later reads S3 with its own keys.
