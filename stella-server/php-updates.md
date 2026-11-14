# PHP updates

**Related:** [`infrastructure.md`](infrastructure.md), [`wordpress-staging.md`](wordpress-staging.md), [`../atlas/dev-instance.md`](../atlas/dev-instance.md), [`../atlas/server.md`](../atlas/server.md).

PHP 2026-10-21 security release: 8.5.12, 8.4.27, 8.3.36, and 8.2.35 fix two critical unauthenticated RCE flaws (CVE-2026-109618, CVE-2026-109619, "PHP2Shell"). All earlier releases are affected; no configuration mitigation exists; exploitation in the wild was reported.

---

## Inventory

Last checked 2026-11-04.

| Where | How installed | Update command | Current version |
|-------|---------------|----------------|-----------------|
| `dominikliss.foxcraft.digital` `wp` | Docker `wordpress:php8.5-apache` (moving minor tag; reported as untagged ID `ce31904fa1e5`) | recreate: `docker compose up -d wp` after pull | 8.5.12. **TODO:** recreate onto the current tag. Live image **OPEN** from atlas.dev-ssh. |
| `dominikliss.foxcraft.digital-ssh` | Docker build `Dockerfile.ssh` (`FROM php:8.5-cli`) | `docker compose up -d --build ssh` | 8.5.12 |
| `hypershape.foxcraft.digital` `wp` | Docker `wordpress:php8.5-apache` (moving minor tag) | `docker compose pull wp` then `up -d wp` | 8.5.12 (was 8.4 until 2026-11-04) |
| `hypershape.foxcraft.digital-ssh` | Shared image `stella/ssh-php:8.5` (`pull_policy: never`) | rebuild the tag, then `docker compose up -d ssh` — [Shared SSH image](#shared-ssh-image-stellassh-php) | 8.5.12 |
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

## WordPress staging on Stella (`hypershape`)

Done 2026-11-04 together with the shared-image switch. Was `wordpress:php8.4-apache` / `Dockerfile.ssh` (`php:8.4-cli`).

1. DB dump first: `/root/hypershape-before-adopt.sql`
2. New compose using `wordpress:php8.5-apache` + `stella/ssh-php:8.5` (backup `docker-compose.yml.bak-20261104`)
3. `docker compose pull wp && docker compose up -d wp ssh`

Verified 2026-11-04: PHP 8.5.12 in `wp` and `-ssh`; site returns HTTP/2 200; no fatals; `getfacl html` still shows `flags: -s-` and `mask::rwx`.

**TODO (Dominik):** delete `/root/hypershape-before-adopt.sql` and `docker-compose.yml.bak-20261104` once the site is confirmed fine.

Details: [`wordpress-staging.md`](wordpress-staging.md#migration-of-hypershape-2026-11-04).

---

## Shared SSH image (`stella/ssh-php`)

Dockerfile (`images/ssh-php/Dockerfile`): after the apt layer, `RUN docker-php-ext-install mysqli pdo_mysql` (backup `Dockerfile.bak-20261114`). The apt layer ends with `&& rm -f /etc/ssh/ssh_host_*` so host keys are not baked into the image (backup `Dockerfile.bak-20261114b`). Needed so WP-CLI in the throwaway install container (and `wp db` over SSH) can talk to MariaDB, and so each stack gets unique host keys from the entrypoint. **Both** `stella/ssh-php:8.4` and `:8.5` must be rebuilt after a Dockerfile or `entrypoint.sh` change, plus a recreate/restart of each ssh container that should pick it up (backup `entrypoint.sh.bak-20261114`, both tags rebuilt 2026-11-14).

A **patch** update needs a rebuild of each local tag, then a recreate of every stack that uses it (running containers keep the old image until then):

```bash
docker build --pull --build-arg PHP_VERSION=8.5 -t stella/ssh-php:8.5 /opt/services/stack-api/images/ssh-php
docker build --pull --build-arg PHP_VERSION=8.4 -t stella/ssh-php:8.4 /opt/services/stack-api/images/ssh-php
cd /opt/apps/wordpress/hypershape.foxcraft.digital
docker compose up -d ssh
```

Rebuilt both tags 2026-11-14 (mysqli, then `/etc/environment`, then host-key removal); verified `php -m` lists `mysqli` and `pdo_mysql` in both, and the image has no `/etc/ssh/ssh_host_*`. Sandbox ssh was recreated (re-render) and its host keys regenerated. hypershape is still on the older image (pre-mysqli, shared build host key). Recreating remaining ssh containers is optional for mysqli — only needed if SSH users run `wp db` there — but hypershape still needs a host-key regenerate together with Edison's and the Mac's `known_hosts`. See [`stack-api.md`](stack-api.md#shared-ssh-image-stellassh-phpversion).

A **minor** bump is a new tag (e.g. `stella/ssh-php:8.6`), not an overwrite of `8.5`. Keep `wp` and the SSH image on the same minor.

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
