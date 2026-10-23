# wp-staging-api (automated WordPress staging from Atlas backups)

**Status:** code in the Atlas repo under `stella/wp-staging-api/`. **Not deployed yet** (2026-10-22). Planned URL: `https://stella-wp-staging.foxcraft.digital`.

Atlas → WP Sites → a completed **full** backup → **Create staging**. Stella pulls the encrypted backup from Atlas, decrypts it, and builds `https://<site-label>.staging.foxcraft.digital` (e.g. `client.com` → `client-com.staging.foxcraft.digital`). A new build for the same site replaces the old one.

Automates the manual steps in [`wordpress-staging.md`](wordpress-staging.md#migrating-an-existing-live-site-into-staging). Backup format: [`../atlas/atlas-connect/backups.md`](../atlas/atlas-connect/backups.md#who-decrypts).

---

## Keys

| Material | Where it is |
|---|---|
| Fleet **private** key | Only in the operator's browser tab, long enough to unwrap one backup's data key. Never sent to Atlas or Stella |
| Per-backup **data key** (32 bytes) | Browser → Atlas `POST …/staging` → Stella `POST /stagings` in the same request. Atlas does not store, log, or queue it. Stella keeps it in a `bytearray` in the build thread and zeroes it when decryption ends (success or failure). Python may still have short-lived immutable copies in memory |
| S3 credentials | Never on Stella. Stella fetches the `.atlas.tar` through a signed Atlas link (default **3 h**, `STELLA_WP_STAGING_LINK_HOURS`) |

The manifest hash Atlas recorded when the backup finished is sent along. Stella refuses an archive whose `manifest.json` does not match it, and every object must be listed in the manifest with the same size before it is decrypted (AEAD with the S3 key as associated data).

---

## API

Same signed-request scheme as [`deploy-api.md`](deploy-api.md): `ssh-keygen -Y verify`, principal `ddashboard`, namespace `deploy`, 60 s window, replay cache. The payload also hashes the body, so the request cannot be swapped:

```
payload = "<action>:<unix_ts>:<sha256(request)>"
POST body = {"payload": …, "signature": …, "request": "<JSON string>"}
```

| Path | Action | Request | Response |
|---|---|---|---|
| `GET /health` | — | — | `{status: "ok"}` |
| `POST /stagings` | `wp-staging-create` | `name, site_domain, site_url, table_prefix, archive_url, archive_bytes, manifest_sha256, data_key` | `{job_id, status, url}`; 409 if a build for `name` runs |
| `POST /jobs/status` | `wp-staging-status` | `job_id` | `{status: queued\|running\|success\|failed\|unknown, step, log[], url, error}` |
| `POST /stagings/delete` | `wp-staging-delete` | `name` | `{deleted, log[]}` |

Jobs live in memory; one build runs at a time. After a container restart, Atlas sees `unknown` and marks the staging as **error** (the site itself may still be there; rebuild or delete it).

Atlas polls every 10 s (`PollWpStagingJob`, queue `backups`) for up to 4 h.

---

## What a build does

1. Disk check: free space ≥ 2.5 × archive size (`STAGING_DISK_HEADROOM`).
2. `docker compose down -v` and delete `/opt/apps/wordpress/<name>.staging.foxcraft.digital/` if present.
3. New folder with `.env` (`chmod 600`, random DB passwords) and `docker-compose.yml`: `wpstg-<name>` (`wordpress:php8.4-apache`, `edge` + `internal`) and `wpstg-<name>-db` (`mariadb:11`, `internal` only). Labels `backup.exclude=true`, `health.enable=false`, `atlas.wp-staging=<name>`. `WP_HOME`/`WP_SITEURL` = staging URL, `WP_ENVIRONMENT_TYPE=staging`, `DISABLE_WP_CRON`, `WP_CACHE=false`. No SSH container.
4. Stream + decrypt the archive: `files/…` → `html/` (skipping `files/_other/` and the live `wp-config.php`; the image's env-driven config is used with the backup's table prefix). `db/` parts → `database.sql` with `CREATE DATABASE` / `USE` lines dropped and `utf8mb4_0900_*` collations mapped to MariaDB ones.
5. MU-plugin `atlas-staging-guard.php`: `pre_wp_mail` returns false (**no outgoing mail**), `noindex`, `blog_public=0`.
6. `.htaccess`: adds `RewriteCond %{HTTP:X-Forwarded-Proto} !https` above an HTTPS-forcing rule (Caddy loop gotcha).
7. `chown -R 33:33 html`, `compose up -d`, wait for MariaDB, import `database.sql`, delete it.
8. WP-CLI (`wordpress:cli-php8.4` via `docker run --volumes-from … --network container:…`, `php -d memory_limit=512M … --skip-plugins --skip-themes`): search-replace `https://<live host>`, `http://<live host>`, `/<live host>` with `--all-tables-with-prefix --skip-columns=guid` (bare domain not replaced, so email addresses stay).
9. Deactivate `atlas-connect`, `mainwp-child`, `wp-mail-smtp`, `mailpoet`, `post-smtp`, `fluent-smtp`; `cache flush`.
10. Local HTTP check inside the container.

The job log (last 400 lines) is shown in Atlas. Passwords in logged commands are masked.

---

## Install on Stella

1. Copy `stella/wp-staging-api/` from the Atlas repo to `/opt/services/wp-staging-api/`.
2. `allowed_signers` in that folder: the same `ddashboard ssh-ed25519 …` line as `/opt/services/deploy-api/allowed_signers`.
3. Merge `docker-compose.service.yml` into `/opt/services/docker-compose.yml`, then `docker compose up -d --build wp-staging-api` from `/opt/services`.
4. `docker pull wordpress:cli-php8.4` (**TODO (Dominik):** confirm the tag exists; otherwise set `STAGING_CLI_IMAGE=wordpress:cli`).
5. DNS: wildcard `*.staging.foxcraft.digital` A → `95.217.144.93`, plus `stella-wp-staging.foxcraft.digital`.
6. Caddy (then `docker compose restart caddy`):

```caddyfile
stella-wp-staging.foxcraft.digital {
    tls {
        issuer acme {
            email projects@foxcraft.digital
            dir https://acme-v02.api.letsencrypt.org/directory
            dns hetzner {env.HETZNER_API_TOKEN}
        }
    }
    reverse_proxy wp-staging-api:8000
}

*.staging.foxcraft.digital {
    tls {
        issuer acme {
            email projects@foxcraft.digital
            dir https://acme-v02.api.letsencrypt.org/directory
            dns hetzner {env.HETZNER_API_TOKEN}
        }
    }
    # Optional second layer on top of the DOCKER-USER IP allowlist:
    # basic_auth { staging <bcrypt hash from `caddy hash-password`> }
    reverse_proxy wpstg-{labels.3}:80
}
```

`{labels.3}` is the first label of `client-com.staging.foxcraft.digital` (labels count from the right). A wildcard certificate needs DNS-01, which the Hetzner DNS plugin already does.

7. Atlas `.env`: `STELLA_WP_STAGING_API_URL=https://stella-wp-staging.foxcraft.digital` (also `STELLA_WP_STAGING_DOMAIN`, `STELLA_WP_STAGING_LINK_HOURS`). Stella's IP `95.217.144.93` is already in Atlas's `ATLAS_ALLOWED_IPS`, which the archive link needs.

Check: `curl -s https://stella-wp-staging.foxcraft.digital/health`.

---

## Limits and risks

- The service has the Docker socket and writes `/opt/apps/wordpress` → root-equivalent on Stella. Signature + IP allowlist are the only gates.
- Long archive downloads go through Atlas PHP-FPM / proxy timeouts. **UNVERIFIED** on production for multi-GB backups.
- Staging sites are not in the restic backup and not monitored by health-api (by design).
- Plugins that send mail through their own API (not `wp_mail`) or call home with live licences are not caught. Review per site.
- Only one build at a time; a second request for another site waits for the lock.
