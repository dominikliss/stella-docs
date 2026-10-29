# Atlas Connect — version reporting and plugin updates

Atlas updates the plugin on every managed site without SSH and without giving sites GitHub access. The site never calls out: Atlas calls the site and sends the update package inside the request.

Contract index, auth, and IP allowlist: [`README.md`](README.md).

---

## Flow

```
Atlas  git tag 1.0.33 → build atlas-connect.zip (single top-level folder atlas-connect/)
Atlas  sign  "atlas-connect-update\n1.0.33\n{sha256(zip)}"  with the Ed25519 private key
Atlas  → POST /atlas-connect/v1/info            ← plugin_version, update.supported, max_package_bytes, jobs_running
Atlas  → POST /atlas-connect/v1/plugin/update   { version, sha256, signature, package: base64(zip) }
Site   verifies hash + signature against config.php update.public_key, checks the zip, runs Plugin_Upgrader
Atlas  → POST /atlas-connect/v1/info            ← plugin_version = 1.0.33 (this request also runs the DB upgrade hook)
```

Both routes follow the plugin conventions: `POST`, JSON body, `secret` in the body, `atlas_secret_authentication()` (SSL → IP → secret). Auth failures are WordPress's `401 rest_forbidden`; a `GET` is `404 rest_no_route`.

---

## Release rule

**git tag = `Version:` header in `atlas-connect.php`.** Atlas reads the tags to decide what "latest" is.

- The header is the only place the version is written. `ATLAS_CONNECT_VERSION` is read from it once when the plugin loads (`get_file_data( __FILE__ )`).
- This repo's tags have no `v` prefix (`1.0.32`). Atlas should strip a leading `v` if one ever appears; the CI check accepts both.
- `tests/release-test.php` (GitHub Actions, every push and tag) fails when the header is not `X.Y.Z`, when a tagged commit's tag differs from the header, when `ip_whitelist` lacks `2.31.42.13` or `95.217.144.93`, or when `update.public_key` is neither empty nor 64 hex. Don't let Atlas offer a tag whose CI run failed.

---

## Signing key

`config.php`:

```php
'update' => [
    'public_key' => '', // Ed25519 public key, 64 hex chars (32 bytes)
],
```

- **Ed25519, for signatures.** It is not `backup.public_key`, which is X25519 and used for backup encryption. Never reuse one for the other.
- Only Atlas holds the private key (64-byte libsodium secret key). Keep it in the Atlas vault, never in a repo.
- While `public_key` is empty, `/plugin/update` refuses every request with `409 update_disabled` and `/info` reports `update.public_key_configured: false`. That is the shipped state until Atlas provides the key.
- Rotating the key is a plugin release (code in `config.php`), pushed with the old key still in place.

Generate once (PHP with sodium):

```php
$kp = sodium_crypto_sign_keypair();
echo bin2hex( sodium_crypto_sign_publickey( $kp ) ), "\n";  // → config.php update.public_key
echo bin2hex( sodium_crypto_sign_secretkey( $kp ) ), "\n";  // → Atlas vault only
```

---

## POST /info

Body: `{ "secret": "…" }`. It does no work and changes nothing.

```json
{
  "plugin_version": "1.0.33",
  "db_version": "1.2",
  "wp_version": "6.8.1",
  "php_version": "8.3.12",
  "multisite": false,
  "environment_type": "production",
  "sodium": true,
  "require_ssl": true,
  "update": {
    "supported": true,
    "public_key_configured": true,
    "filesystem_method": "direct",
    "file_mods_allowed": true,
    "plugin_dir_writable": true,
    "max_package_bytes": 8388608
  },
  "jobs_running": { "backup": false, "scan": false }
}
```

| Field | Source |
|---|---|
| `plugin_version` | `ATLAS_CONNECT_VERSION` — the `Version:` header, read when the running code loaded |
| `db_version` | option `atlas_db_version` (`""` if unset) |
| `wp_version` | `get_bloginfo( 'version' )` |
| `php_version` | `PHP_VERSION` |
| `multisite` | `is_multisite()` |
| `environment_type` | `wp_get_environment_type()` |
| `sodium` | libsodium available |
| `require_ssl` | the site's effective SSL requirement |
| `update.public_key_configured` | `update.public_key` is 64 hex |
| `update.filesystem_method` | `get_filesystem_method()` |
| `update.file_mods_allowed` | `wp_is_file_mod_allowed( 'atlas_connect_update' )` — `false` under `DISALLOW_FILE_MODS` |
| `update.plugin_dir_writable` | `wp-content/plugins/atlas-connect` and `wp-content/plugins` both writable |
| `update.max_package_bytes` | Largest **decoded** zip accepted: `min( 8 MB, post_max_size × 0.7 )` (base64 + JSON overhead). With PHP's default `post_max_size = 8M` that is `5872025` |
| `update.supported` | All of: key configured, method `direct`, file mods allowed, plugin dir writable, sodium — plus PHP `zip` (`ZipArchive`) and the plugin living in a folder named `atlas-connect` |
| `jobs_running` | A backup / scan job is currently `running` on this site |

Never returned: the secret, its hash or hint, file paths, S3 data, the IP lists, any key.

Atlas: only push when `update.supported` is `true`, the zip is ≤ `max_package_bytes`, and both `jobs_running` flags are `false`. A body over `post_max_size` is dropped by PHP before WordPress sees it and comes back as `401`, so check the size first.

---

## POST /plugin/update

```json
{
  "secret": "…",
  "version": "1.0.33",
  "sha256": "<64 lowercase hex, sha256 of the decoded zip bytes>",
  "signature": "<128 hex, Ed25519 detached signature>",
  "package": "<base64 of the zip>"
}
```

### Signed message

```
atlas-connect-update\n{version}\n{sha256}
```

LF line breaks, **no trailing newline**, `sha256` in lowercase hex. The signature binds the version to the exact bytes, so nobody can relabel an old signed zip as a new version.

### Checks

In this order; the first failure wins. Every error is `{ "error": "<human text>", "code": "<code>" }`.

| # | Check | HTTP | `code` |
|---|---|---|---|
| 1 | `update.public_key` configured, sodium and `ZipArchive` available, plugin folder is `atlas-connect` | 409 | `update_disabled` |
| 2 | `wp_is_file_mod_allowed()`, `get_filesystem_method() === 'direct'` (the method is named in `error`), plugin dir writable | 409 | `filesystem_not_writable` |
| 3 | No other update in progress. Lock: option `atlas_update_lock` = timestamp, stale after 10 minutes | 409 | `update_in_progress` |
| 4 | No backup or scan job running | 409 | `job_running` |
| 5 | `version` matches `^\d+\.\d+\.\d+$` | 400 | `invalid_version` |
| 6 | `version >= ATLAS_CONNECT_VERSION` (same version = reinstall) | 409 | `downgrade_refused` |
| 7 | `sha256` 64 lowercase hex, `signature` 128 hex, `package` non-empty strict base64 | 400 | `invalid_request` |
| 8 | Decoded size ≤ `max_package_bytes` | 413 | `package_too_large` |
| 9 | `sha256` equals `sha256( decoded zip )` | 400 | `hash_mismatch` |
| 10 | Ed25519 signature over the signed message verifies against `update.public_key` | 400 | `bad_signature` |
| 11 | Zip opens; every entry is under `atlas-connect/` with no `..`, absolute or backslash paths; contains `atlas-connect/atlas-connect.php`; that file's `Version:` header equals `version` | 400 | `invalid_package` |
| — | `Plugin_Upgrader` failed (skin error messages in `error`), or reactivation failed | 500 | `install_failed` |

The signature (10) is checked before the structure (11), so the site never opens an unsigned zip with `ZipArchive`.

### Downgrade policy

Downgrades are always refused. There is no override flag: a stolen secret must not be able to install an older, vulnerable signed build. **To roll back, release a higher version** with the old code.

### Install

1. The zip is written to `wp_tempnam()`, `WP_Filesystem()` is initialised.
2. `Plugin_Upgrader( new WP_Ajax_Upgrader_Skin() )->install( $tmp, [ 'overwrite_package' => true ] )` replaces `wp-content/plugins/atlas-connect/`.
3. `wp_clean_plugins_cache()`, opcache invalidated for the plugin dir.
4. If the plugin is no longer active it is reactivated silently (network-wide if it was network-active before).
5. The temp file is deleted and the lock released in every case.

The DB upgrade does **not** run inline. The existing `plugins_loaded` hook runs it on the next request when `atlas_db_version` changed — Atlas's follow-up `/info` is that request.

Success `200`:

```json
{ "updated": true, "previous_version": "1.0.32", "version": "1.0.33", "installed_version": "1.0.33" }
```

`installed_version` is read with `get_plugin_data()` from the file now on disk. `ATLAS_CONNECT_VERSION` was read from the header when this request started, so it still holds the old version — the old code is what is running. `/info` afterwards reports the new one.

The site never logs `package`, `secret`, or `signature`.

### Atlas HTTP mapping

| Site | Atlas does |
|---|---|
| `200` | Call `/info`; expect `plugin_version === version` |
| `409 update_disabled` / `filesystem_not_writable` | Mark the site "manual updates only"; show `error` |
| `409 update_in_progress` / `job_running` | Retry later |
| `409 downgrade_refused` | Site is already newer — refresh `/info` |
| `400` / `413` | Bug in Atlas's build or signing — do not retry the same package |
| `500 install_failed` | Alert; check the site (`/info`, WP Admin) |
| `401` | Auth (secret / SSL / IP), or body over `post_max_size` |
| `404 rest_no_route` | Plugin older than this release — needs the one-time manual install |

---

## Building the package (Atlas)

- From the git tag, single top-level folder `atlas-connect/`.
- Strip `.git`, `.github`, `.cursor`, `docs`, `tests`, and the submodule folders `stella-docs` / `dev-docs`.
- `sha256` over the zip bytes, sign the message above, base64 the zip.

```php
$zip = file_get_contents( $path );
$sha = hash( 'sha256', $zip );
$sig = sodium_crypto_sign_detached( "atlas-connect-update\n{$version}\n{$sha}", hex2bin( $vaultSecretKeyHex ) );
$body = [ 'secret' => $siteSecret, 'version' => $version, 'sha256' => $sha, 'signature' => bin2hex( $sig ), 'package' => base64_encode( $zip ) ];
```

---

## First manual install

Sites on versions before these routes existed answer `404 rest_no_route`. They need **this release installed by hand once** — WP Admin → Plugins → Add New → Upload (the zip from **Download Atlas Connect**), or the existing SSH/rsync deployment. After that, Atlas can push.

Sites deployed with rsync may also have a `tests/` folder. Everything in it exits unless run from the PHP CLI (and `tests/.htaccess` denies Apache), so requesting it over HTTP does nothing. The first pushed update replaces the whole plugin folder, so it goes away.

---

## What does not exist

- No update checks from the site: no `pre_set_site_transient_update_plugins` hook, no update server, no GitHub calls.
- No route to read the key or change it.
