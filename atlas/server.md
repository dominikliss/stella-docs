# Atlas production host

Hetzner Cloud server **`atlas`**. Moved off the Hetzner managed host (`dev.atlas.foxcraft.digital`, user `foxcrr`) on 2026-10-10. Not containerized. Separate Hetzner Cloud project from Edison. **TODO (Dominik):** name of the Atlas Cloud project.

**Related:** [`README.md`](README.md) (product), [`architecture.md`](architecture.md) (code map), Stella firewall [`../stella-server/infrastructure.md`](../stella-server/infrastructure.md).

Admin work as **root** via `ssh atlas`. App runs as Linux user **`atlas`**.

---

## Server

| | |
|---|---|
| **Name** | `atlas` |
| **Type** | CX33 — 4 vCPU / 8 GB |
| **OS** | Ubuntu 26.04 |
| **IP** | `2.31.42.13` |
| **URL** | `https://atlas.foxcraft.digital` (replaces `https://dev.atlas.foxcraft.digital`) |
| **App path** | `/var/www/atlas` |
| **App user** | `atlas` |
| **Repo** | `git@github.com:foxcraftdigital/atlas.git` — **not** `foxcraft/atlas` (that repo did not exist before 2026-10-10) |
| **Submodules** | `docs/stella-docs` and `docs/dev-docs` are **not** checked out on production |
| **Backups** | Hetzner Cloud backups **enabled** |
| **ufw / fail2ban** | none — nothing is exposed beyond the allowlisted IP |

---

## Stack

| Layer | What runs |
|-------|-----------|
| PHP | 8.5.4-FPM, pool `atlas`, socket `/run/php/atlas.sock`, pool file `/etc/php/8.5/fpm/pool.d/atlas.conf` |
| App | Laravel 12.61 |
| MySQL | 8.4, database `atlas_prod`, user `atlas`, localhost only |
| Redis | localhost only — **cache only** (not the deploy queue) |
| Node | 22 / npm 9.2 (Vite build) |
| Caddy | 2.11.7, built with xcaddy + `github.com/caddy-dns/hetzner/v2`, binary `/usr/bin/caddy`, `apt-mark hold caddy` |

Old DB name on the managed host was `dev_atlas_foxcraft_digital` (docs previously said `atlas_dev`). Migrated via phpMyAdmin dump (MariaDB 10.11 → MySQL 8.4) into `atlas_prod`, 27 tables, all migrations Ran. A dump from MariaDB imports into MySQL 8.4 when it has no `DEFINER` and no `CREATE DATABASE` / `USE` lines.

---

## Caddy

| Path | Role |
|------|------|
| `/etc/caddy/Caddyfile` | Site config |
| `/etc/caddy/caddy.env` | Hetzner DNS token (`chmod 640`, owner `root:caddy`) — do not print |
| `/etc/systemd/system/caddy.service.d/override.conf` | `EnvironmentFile` + `ExecStart` **without** `--environ` |

Ubuntu’s `caddy.service` uses `--environ`, which writes the Hetzner API token into the journal. The override removes `--environ`. Vacuum the journal if it already leaked.

Site block follows the standard Stella template (DNS-01, `dir` pinned to Let's Encrypt production). Root `/var/www/atlas/public`, `php_fastcgi unix//run/php/atlas.sock`. **No** `@blocked` matcher — Hetzner firewall + `ATLAS_ALLOWED_IPS` do the filtering.

```caddyfile
atlas.foxcraft.digital {
    tls {
        issuer acme {
            email projects@foxcraft.digital
            dir https://acme-v02.api.letsencrypt.org/directory
            dns hetzner {env.HETZNER_API_TOKEN}
        }
    }
    root * /var/www/atlas/public
    php_fastcgi unix//run/php/atlas.sock
    file_server
}
```

### Rebuild Caddy

Ubuntu apt Caddy is 2.6.2 with no Hetzner plugin. caddyserver.com’s download API returned 400 for the `hetzner/v2` module path. Build with xcaddy (needs `golang-go`), then `apt-mark hold caddy`.

---

## Hetzner Cloud firewall (inbound)

| Rule | Source |
|------|--------|
| TCP/22, TCP/80, TCP/443 | `194.126.177.181` only |
| ICMP | `194.126.177.181` only |

---

## Queue worker

systemd **`atlas-queue.service`** — replaces cron + `flock` `scripts/queue-deployments.sh`. That script and its lock/log paths (`/usr/home/foxcrr/logs/queue-worker.log`, `/usr/home/foxcrr/tmp/atlas-deploy-queue.lock`) **no longer apply** on this host.

- `User=atlas`
- `WorkingDirectory=/var/www/atlas`
- `php artisan queue:work database --queue=deployments --sleep=3 --timeout=7200 --max-time=3500`
- `Restart=always`

Logs: `journalctl -u atlas-queue`.

---

## Scheduler

`/etc/cron.d/atlas` runs `php artisan schedule:run` every minute as user `atlas`.

| Schedule | Job | Notes |
|----------|-----|--------|
| `*/5` | `stella:check-health` | Verified 2026-10-10 via cron |
| `*/5` | `wp-sites:check-uptime` | |
| every minute | `wp-sites:drive-backups` | Not documented before. Backups feature still in development; backup driver not finished |

---

## `.env` (production)

Do not write secret values here.

| Key | Production |
|-----|------------|
| `APP_ENV` | `production` |
| `APP_DEBUG` | `false` |
| `APP_URL` | `https://atlas.foxcraft.digital` |
| `DB_HOST` | `127.0.0.1` |
| `DB_DATABASE` | `atlas_prod` |
| `DEPLOY_SSH_KEY_PATH` | `/home/atlas/.ssh/id_ed25519` |

`APP_KEY` was carried over unchanged from the old instance (needed for encrypted `app_settings` values). `app_settings.deploy_ssh_key_path` was updated to `/home/atlas/.ssh/id_ed25519` (the DB value overrides `.env`).

`ATLAS_ALLOWED_IPS` = `194.126.177.181`, `23.88.90.12` (managed host), `95.217.144.93` (Stella). **TODO (Dominik):** remove `23.88.90.12` unless something on the managed host really calls Atlas; reason for each entry unverified.

---

## SSH key

Atlas reuses the same keypair as the old managed-host Atlas (principal `ddashboard` in Stella `allowed_signers`, key comment `atlas-prod`), at `/home/atlas/.ssh/id_ed25519`. Deliberate: avoid updating `authorized_keys` on the 28 deploy servers (`deploy_servers`: `label`, `ssh_host`, `ssh_port`, `ssh_user`, `site_url`).

**Tradeoff:** the managed host still holds the same key, so it can still deploy/sign. Rotation was considered and deferred.

---

## Deploy code

As user `atlas` in `/var/www/atlas`:

```bash
git pull
composer install --no-dev --optimize-autoloader
npm ci --legacy-peer-deps && npm run build
php artisan migrate --force
php artisan config:cache
```

Then restart the queue: `systemctl restart atlas-queue`.

`npm ci` fails with `ERESOLVE` (`@types/node` ^18 vs vite 7 peer `^20.19 \|\| >=22.12`). Workaround `--legacy-peer-deps`. Real fix in the Atlas repo: `@types/node@^22`. **TODO (Dominik):** fix in repo.

---

## Verified 2026-10-10 (from this host)

- Login via ddashboard
- Signed health-api check (status ok, 21 services)
- Signed deploy-api calls
- Several `ssh_rsync` deployments
- Scheduled health checks via cron

---

## Old instance (`dev.atlas.foxcraft.digital`)

Still on the managed host. Its two cron jobs (`queue-deployments.sh`, `schedule:run`) are to be deactivated. **TODO (Dominik):** confirm deactivated and decide when to retire the instance.

---

## Gotchas (2026-10-10)

1. A new A record may already exist pointing elsewhere (`atlas.foxcraft.digital` initially pointed at `23.88.90.12`). DNS-01 does not care, but traffic would go to the wrong server.
2. Staging-CA fallback happened again on first issuance; a full Caddy restart after DNS was live fixed it (same pattern as `stella-health-api` — see [`../stella-server/infrastructure.md`](../stella-server/infrastructure.md) Issue 3).
3. Ubuntu’s `caddy.service` `--environ` prints the Hetzner token into the journal. Override `ExecStart`; vacuum the journal if it already leaked.
4. A leftover `umask 077` in the shell created `/var/www` as `700`, so user `atlas` could not write. Set `umask 022`, `chmod 755`.
5. Repo `.gitignore` check before first push: no `.env`, `vendor`, `node_modules`, storage logs in the repo (verified clean).
6. `npm ci` ERESOLVE — see deploy section above.
7. Do not use apt Caddy for DNS-01; build with xcaddy.
8. MariaDB phpMyAdmin dump → MySQL 8.4: fine if no `DEFINER` and no `CREATE DATABASE` / `USE`.
