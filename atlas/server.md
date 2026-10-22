# Atlas production host

Hetzner Cloud server **`atlas`**. Moved off the Hetzner managed host (`dev.atlas.foxcraft.digital`, user `foxcrr`) on 2026-10-10. Not containerized. Same Hetzner Cloud project as Edison. **TODO (Dominik):** name of that Cloud project.

**Related:** [`README.md`](README.md) (product), [`architecture.md`](architecture.md) (code map), [`host-api.md`](host-api.md) (Atlas host API), Stella firewall [`../stella-server/infrastructure.md`](../stella-server/infrastructure.md).

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
| **App user** | `atlas` (uid/gid 1000). Run git / artisan / npm as `atlas` (`sudo -u atlas …`), never as root (dubious ownership; root has no GitHub key). |
| **DNS** | A record `atlas` in zone `567656` → `2.31.42.13`. Let's Encrypt production via DNS-01. |
| **Repo** | `git@github.com:foxcraftdigital/atlas.git` — **not** `foxcraft/atlas` (that repo did not exist before 2026-10-10) |
| **Submodules** | `docs/stella-docs` and `docs/dev-docs` are **not** checked out on production |
| **Backups** | Hetzner Cloud backups **enabled** |
| **ufw / fail2ban** | none — nothing is exposed beyond the allowlisted IP |

---

## Stack

| Layer | What runs |
|-------|-----------|
| PHP | 8.5.4-FPM, Ubuntu `8.5.4-0ubuntu1.5` (includes CVE-2026-109618/109619 backports), pool `atlas`, socket `/run/php/atlas.sock`, pool file `/etc/php/8.5/fpm/pool.d/atlas.conf` |
| App | Laravel 12.61 |
| MySQL | 8.4, database `atlas_prod`, user `atlas`, localhost only |
| Redis | 8.0, localhost only — **cache only** (not the deploy queue) |
| Node | 22 / npm 9.2 (Vite build) |
| Caddy | 2.11.7, built with xcaddy + `github.com/caddy-dns/hetzner/v2`, binary `/usr/bin/caddy`, `apt-mark hold caddy` |
| Host API | Python/FastAPI, user `atlasapi`, systemd `atlas-api`, `127.0.0.1:8090` — see [`host-api.md`](host-api.md) |

Old DB name on the managed host was `dev_atlas_foxcraft_digital` (docs previously said `atlas_dev`). Migrated via phpMyAdmin dump (MariaDB 10.11 → MySQL 8.4) into `atlas_prod`, 27 tables, all migrations Ran. Import used `mysql --default-character-set=utf8mb4`. A dump from MariaDB imports into MySQL 8.4 when it has no `DEFINER` and no `CREATE DATABASE` / `USE` lines.

### PHP security updates

Do not judge by `php -v` alone (it correctly shows 8.5.4). `apt changelog php8.5-fpm` returned 404 for this version (changelogs.ubuntu.com). Use `apt-cache policy php8.5-fpm` plus `/usr/share/doc/php8.5-common/changelog.Debian.gz` — that file lists `SECURITY UPDATE: Remote RCE via state corruption issue` with `CVE-2026-109618_109619.patch`.

No Sury repo (`packages.sury.org`) on prod. Ubuntu security backports are enough. Sury would be the way to reach 8.5.12 exactly. Not used.

Full inventory and other hosts: [`../stella-server/php-updates.md`](../stella-server/php-updates.md).

Verified 2026-10-23: FPM main process (`/usr/sbin/php-fpm8.5`) and `atlas-queue` (`/usr/bin/php8.5`) run without `(deleted)` in `/proc/<pid>/exe`.

Also 2026-10-22: `apt upgrade` (18 packages: systemd 259.5 → 259.9, udev, cloud-init, snapd, open-iscsi, open-vm-tools, …). No reboot required. Services active: `mysql`, `redis-server`, `php8.5-fpm`, `caddy`, `atlas-queue`, `atlas-api`. `systemctl --failed` empty. `needrestart` restarted `redis-server` and `php8.5-fpm`. Caddy stays on the xcaddy build (`apt-mark hold`).

```bash
# confirm no customer deploy is running
apt update
apt install --only-upgrade $(dpkg -l | awk '/^ii  php8\.5-/{print $2}')
systemctl reload php8.5-fpm
sudo -u atlas php /var/www/atlas/artisan queue:restart
# then: readlink /proc/<fpm-pid>/exe and /proc/<queue-pid>/exe — no "(deleted)"
```

`queue:restart` is graceful and replaces `systemctl restart atlas-queue` for this update.

---

## Caddy

| Path | Role |
|------|------|
| `/etc/caddy/Caddyfile` | Site config |
| `/etc/caddy/caddy.env` | Hetzner DNS token (`chmod 640`, owner `root:caddy`) — do not print |
| `/etc/systemd/system/caddy.service.d/override.conf` | `EnvironmentFile` + `ExecStart` **without** `--environ` |

Ubuntu’s `caddy.service` uses `--environ`, which writes the Hetzner API token into the journal. The override removes `--environ`. Vacuum the journal if it already leaked.

Site block follows the standard Stella template (DNS-01, `dir` pinned to Let's Encrypt production). Root `/var/www/atlas/public`, `php_fastcgi unix//run/php/atlas.sock`. **No** `@blocked` matcher — Hetzner firewall + `ATLAS_ALLOWED_IPS` do the filtering.

A second site block `api.atlas.foxcraft.digital` fronts the host API (`atlas-api` on `127.0.0.1:8090`). See [`host-api.md`](host-api.md). After `rm` + recreate of `/etc/caddy/Caddyfile`, `chmod 644 /etc/caddy/Caddyfile` is required (`atlasapi` reads it).

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
| TCP/443 | `95.217.144.93` (Stella, Atlas dev → host API) |
| ICMP | `194.126.177.181` only |

---

## Queue worker

systemd **`atlas-queue.service`** — replaces cron + `flock` `scripts/queue-deployments.sh`. That script and its lock/log paths (`/usr/home/foxcrr/logs/queue-worker.log`, `/usr/home/foxcrr/tmp/atlas-deploy-queue.lock`) **no longer apply** on this host.

- `User=atlas`
- `WorkingDirectory=/var/www/atlas`
- `php artisan queue:work database --queue=deployments --sleep=3 --timeout=7200 --max-time=3500`
- `Restart=always`

`restart_count` rises ~24/day because of `--max-time=3500` (clean exit, `Restart=always`). Do not alert on it.

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
| `ATLAS_DDASHBOARD_URL` | `https://data3.dominikliss.com/wp-json` |

`APP_KEY` was carried over unchanged from the old instance (needed for encrypted `app_settings` values). `app_settings.deploy_ssh_key_path` was updated to `/home/atlas/.ssh/id_ed25519` (the DB value overrides `.env`).

`ATLAS_ALLOWED_IPS` = `194.126.177.181`, `23.88.90.12` (managed host), `95.217.144.93` (Stella). **TODO (Dominik):** remove `23.88.90.12` unless something on the managed host really calls Atlas; reason for each entry unverified.

---

## SSH key

Atlas reuses the same keypair as the old managed-host Atlas (principal `ddashboard` in Stella `allowed_signers`, key comment `atlas-prod`), at `/home/atlas/.ssh/id_ed25519`. Deliberate: avoid updating `authorized_keys` on the 28 deploy servers (`deploy_servers`: `label`, `ssh_host`, `ssh_port`, `ssh_user`, `site_url`).

**Tradeoff:** the managed host still holds the same key, so it can still deploy/sign. Rotation was considered and deferred.

---

## Deploy code

Current release: **tag `1.0.7`**.

The manual steps below are automated by `/usr/local/bin/atlas-deploy` (ref argument, default `main`, exclusive `flock` on `/var/lock/atlas-deploy.lock`, early exit when HEAD is unchanged). The same script can be triggered from Atlas dev through the host API. See [`host-api.md`](host-api.md). Keep the commands here as the fallback.

| Path | Role |
|------|------|
| `/usr/local/bin/atlas-deploy` | Deploy script (`root:root`, mode 750) |
| `/etc/sudoers.d/atlas-api` | `atlasapi ALL=(root) NOPASSWD: /usr/local/bin/atlas-deploy` (mode 440) |

**Not yet verified** — a deploy that pulls new commits (composer, migrate, npm build, reload, queue restart) started through the API. A no-op run (`already at b087616`) via `POST /system/deploy` from Stella succeeded 2026-10-10. Confirm a real pull before treating the full path as proven.

As **root** on host `atlas`:

```bash
atlas-deploy              # main
atlas-deploy <tag|branch>
```

Script: `/usr/local/bin/atlas-deploy`. Git, artisan, and npm run as user `atlas` (`sudo -u atlas …`), never as root (dubious ownership; root has no GitHub key).

Behavior:

- Ref must match `^[A-Za-z0-9._/-]+$` or the script exits (`invalid ref`).
- Exclusive `flock` on `/var/lock/atlas-deploy.lock`; if another deploy is running it exits (`deploy already running`).
- Fetches tags, checks out the ref, and `git pull --ff-only` when HEAD is a branch.
- Exits early if HEAD did not change.
- Then as `atlas`: `composer install --no-dev --optimize-autoloader`, `php artisan migrate --force`, `php artisan config:clear`, `npm ci --legacy-peer-deps && npm run build`.
- Then as root: `systemctl reload php8.5-fpm` and `systemctl restart atlas-queue`.

```bash
#!/usr/bin/env bash
# /usr/local/bin/atlas-deploy
set -euo pipefail

APP=/var/www/atlas
REF="${1:-main}"

[[ "$REF" =~ ^[A-Za-z0-9._/-]+$ ]] || { echo "invalid ref"; exit 1; }

exec 9>/var/lock/atlas-deploy.lock
flock -n 9 || { echo "deploy already running"; exit 1; }

as_atlas() { sudo -u atlas bash -c "cd $APP && $*"; }

OLD=$(as_atlas "git rev-parse HEAD")

as_atlas "git fetch --tags origin"
as_atlas "git checkout $REF"
as_atlas "if git symbolic-ref -q HEAD >/dev/null; then git pull --ff-only origin $REF; fi"

NEW=$(as_atlas "git rev-parse HEAD")
[ "$OLD" = "$NEW" ] && { echo "already at ${NEW:0:7}"; exit 0; }

as_atlas "composer install --no-dev --optimize-autoloader"
as_atlas "php artisan migrate --force"
as_atlas "php artisan config:clear"
as_atlas "npm ci --legacy-peer-deps && npm run build"

systemctl reload php8.5-fpm
systemctl restart atlas-queue

echo "deployed ${OLD:0:7} -> ${NEW:0:7} ($REF)"
```

`npm ci` fails with `ERESOLVE` (`@types/node` ^18 vs vite 7 peer `^20.19 || >=22.12`). Workaround `--legacy-peer-deps`. Real fix in the Atlas repo: `@types/node@^22`. **TODO (Dominik):** bump to `@types/node@^22`.

### Manual fallback

If the script is unavailable, as user `atlas` in `/var/www/atlas` (`sudo -u atlas …` from root — never as root):

```bash
git pull
php artisan config:clear
npm ci --legacy-peer-deps && npm run build   # public/build is gitignored
```

If `composer.lock` or migrations changed, also (as `atlas`):

```bash
composer install --no-dev --optimize-autoloader
php artisan migrate --force
```

Then as **root**:

```bash
systemctl reload php8.5-fpm
systemctl restart atlas-queue
```

### Repo fixes in tag `1.0.1` (commit `3027079`)

- `config/deploy.php`: `ssh_key_path` default no longer reads `$_SERVER['HOME']` unguarded (`$_SERVER['HOME'] ?? getenv('HOME') ?: '/var/www'`).
- Migration `2026_05_28_120003_create_deploy_run_destinations_table`: `longText` default is `DB::raw("('')")` for MySQL 8.4.

---

## Verified 2026-10-10 (from this host)

- Login via ddashboard
- Signed health-api check (status ok, 21 services — Atlas host-move check). Later the same day, after health-api became dynamic: 20 containers + 7 certs. See [`../stella-server/health-api.md`](../stella-server/health-api.md).
- Signed deploy-api calls
- Several `ssh_rsync` deployments
- Scheduled health checks via cron

---

## Old instance (`dev.atlas.foxcraft.digital`)

Still on the managed host. That instance is to be retired. Its two cron jobs (`queue-deployments.sh`, `schedule:run`) are to be deactivated. **TODO (Dominik):** confirm deactivated and decide when to retire the instance.

The Stella-hosted **dev** instance is a different machine: [`dev-instance.md`](dev-instance.md) (`https://atlas.dev.foxcraft.digital`).

---

## Gotchas (2026-10-10)

1. A new A record may already exist pointing elsewhere (`atlas.foxcraft.digital` initially pointed at `23.88.90.12`). DNS-01 does not care, but traffic would go to the wrong server.
2. Staging-CA fallback happened again on first issuance; a full Caddy restart after DNS was live fixed it (same pattern as `stella-health-api` — see [`../stella-server/infrastructure.md`](../stella-server/infrastructure.md) Issue 3).
3. Ubuntu’s `caddy.service` `--environ` prints the Hetzner token into the journal. Override `ExecStart`; vacuum the journal if it already leaked.
4. A leftover `umask 077` in the shell created `/var/www` as `700`, so user `atlas` could not write. Set `umask 022`, `chmod 755`.
5. Repo `.gitignore` check before first push: no `.env`, `vendor`, `node_modules`, storage logs in the repo (verified clean).
6. `npm ci` ERESOLVE — see deploy section above.
7. Do not use apt Caddy for DNS-01; build with xcaddy.
8. MariaDB phpMyAdmin dump → MySQL 8.4: fine if no `DEFINER` and no `CREATE DATABASE` / `USE`. Import with `mysql --default-character-set=utf8mb4`.
9. Check **both directions** of access: Atlas (`2.31.42.13`) gets **401** from ddashboard, but Stella (`95.217.144.93`) gets **403** from the managed host’s Apache. **TODO (Dominik):** find the IP allowlist rule (probably `.htaccess` under `public_html`) and add Stella if Stella should call ddashboard.
10. Caddy ACME staging fallback on `api.atlas` was caused by missing `propagation_delay` (NXDOMAIN on `_acme-challenge` TXT during secondary validation). Fix: `propagation_delay 90s` + `propagation_timeout 5m` in the `issuer acme` block, then stop/start caddy. See [`host-api.md`](host-api.md).
