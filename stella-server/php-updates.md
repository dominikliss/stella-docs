# PHP updates

**Related:** [`infrastructure.md`](infrastructure.md), [`wordpress-staging.md`](wordpress-staging.md), [`../atlas/dev-instance.md`](../atlas/dev-instance.md), [`../atlas/server.md`](../atlas/server.md).

PHP 2026-10-21 security release: 8.5.12, 8.4.27, 8.3.36, and 8.2.35 fix two critical unauthenticated RCE flaws (CVE-2026-109618, CVE-2026-109619, "PHP2Shell"). All earlier releases are affected; no configuration mitigation exists; exploitation in the wild was reported.

---

## Inventory

Last checked 2026-10-23.

| Where | How installed | Update command | Current version |
|-------|---------------|----------------|-----------------|
| `dominikliss.foxcraft.digital` `wp` | Docker `wordpress:php8.5-apache` (moving minor tag) | `docker compose pull wp` then `up -d wp` | 8.5.12 |
| `dominikliss.foxcraft.digital-ssh` | Docker build `Dockerfile.ssh` (`FROM php:8.5-cli`) | `docker compose up -d --build ssh` | 8.5.12 |
| Atlas dev `app` / `queue` / `scheduler` | Local image `atlasdev-app` (`app` has `build: .`; Dockerfile `FROM php:8.5-apache`; `x-atlas` sets `image: atlasdev-app`) | `docker compose build --pull app` then `up -d app queue scheduler` | 8.5.12 |
| Atlas dev `ssh` | Local build `Dockerfile.ssh` (`php:8.5-cli`) | `docker compose build --pull ssh` then `up -d ssh` | 8.5.12 **UNVERIFIED** (`php -v` not checked after rebuild) |
| Atlas prod (host `atlas`) | Ubuntu `php8.5-cli` / `php8.5-common` / `php8.5-fpm` `8.5.4-0ubuntu1.5` (`resolute-security`) | see [Atlas prod](#atlas-prod-host-atlas) | 8.5.4-0ubuntu1.5 (CVE backports) |
| Managed host `dedi6724` (ddashboard, Foxx Cloud, `dominikliss.com` live) | Hosting-panel PHP | provider update | Operator says updated; exact version **UNVERIFIED** |

**Not PHP:** stella-api, deploy-api, health-api, media-api (Python), imap-sync (Node), Caddy, Ollama, .NET apps.

---

## Rules

- Docker images use **moving minor tags** (`wordpress:php8.5-apache`, `php:8.5-apache`, `php:8.5-cli`). A new patch lands only after `docker compose pull` or `docker compose build --pull`. `php:8.5-cli` can be pinned (`php:8.5.12-cli`) but is not. The `wordpress` image has no exact patch tag.
- Ubuntu hosts: do **not** judge by `php -v` alone. `apt changelog php8.5-fpm` returned 404 for this version (changelogs.ubuntu.com). Use `apt-cache policy php8.5-fpm` plus `/usr/share/doc/php8.5-common/changelog.Debian.gz`.
- Atlas prod: no Sury repo (`packages.sury.org`). Ubuntu security backports are enough. Sury would be the way to reach 8.5.12 exactly. Not used.

---

## WordPress staging on Stella (`dominikliss`)

Done 2026-10-23. Was `wordpress:php8.4-apache` / `php:8.4-cli`.

1. DB dump first: `mariadb-dump … wp > ~/wp-before-php85.sql` (86 MB, `/root` on Stella).
2. Edit `docker-compose.yml` (`wordpress:php8.5-apache`) and `Dockerfile.ssh` (`FROM php:8.5-cli`).
3. `docker compose pull wp`
4. `docker compose up -d --build wp ssh`

Verify:

```bash
docker exec dominikliss.foxcraft.digital php -v
docker exec dominikliss.foxcraft.digital-ssh php -v
docker exec dominikliss.foxcraft.digital php -i | grep -E 'upload_max_filesize|post_max_size'
curl -sI https://dominikliss.foxcraft.digital   # 200
getfacl html | grep -E 'flags|mask'             # flags: -s-  and  mask::rwx
```

Verified 2026-10-23: PHP 8.5.12 in `wp` and `-ssh`; `upload_max_filesize` and `post_max_size` are 64M; site returns 200; no fatals in the Apache log; `getfacl html` still shows `flags: -s-` and `mask::rwx`.

**TODO (Dominik):** delete `/root/wp-before-php85.sql` once the site is confirmed fine.

Upload-limit mount: [`wordpress-staging.md`](wordpress-staging.md#php-limits).

---

## Atlas dev (Stella)

`app`, `queue`, and `scheduler` share the locally built image `atlasdev-app`. `docker compose pull app` fails with "pull access denied" (image is not in a registry). `db` (`mysql:8.4`) is untouched.

Was on 8.5.11. Updated 2026-10-23.

1. Confirm no customer deploy is running on this instance.
2. `docker compose build --pull app`
3. `docker compose up -d app queue scheduler`
4. `docker compose build --pull ssh`
5. `docker compose up -d ssh`

Containers: `atlas.dev.foxcraft.digital`, `-db`, `-queue`, `-scheduler`, `-ssh`.

Verify:

```bash
docker exec atlas.dev.foxcraft.digital php -v
docker exec atlas.dev.foxcraft.digital-ssh php -v
docker ps --filter name=atlas.dev.foxcraft.digital
```

Verified 2026-10-23: PHP 8.5.12 in `app`; all five containers Up; Caddy returns 403 from Stella itself (expected, IP filter). SSH container PHP version: **UNVERIFIED**.

---

## Atlas prod (host `atlas`)

Ubuntu `php8.5-*` `8.5.4-0ubuntu1.5`. The local changelog (`/usr/share/doc/php8.5-common/changelog.Debian.gz`) lists `SECURITY UPDATE: Remote RCE via state corruption issue` with `CVE-2026-109618_109619.patch` and its pre-patches. `php -v` still shows 8.5.4; the CVEs are fixed via backport.

Verified 2026-10-23: FPM main process (`/usr/sbin/php-fpm8.5`) and `atlas-queue` (`/usr/bin/php8.5`) have no `(deleted)` in `/proc/<pid>/exe`.

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
