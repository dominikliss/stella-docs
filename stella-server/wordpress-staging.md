# WordPress Staging Pattern

Established 2026-10-03 with `dominikliss.foxcraft.digital` as the first implementation. This is the host-wide template for WordPress staging on Stella — same role as [`dotnet-app-deployment.md`](dotnet-app-deployment.md) for .NET apps. The live site was imported into this staging on 2026-10-11 (see [Migrating an existing live site into staging](#migrating-an-existing-live-site-into-staging)).

**Related:** [`infrastructure.md`](infrastructure.md) (Caddy, DNS, `DOCKER-USER`), [`dev-ssh-access.md`](dev-ssh-access.md) (SSH variant notes), [`php-updates.md`](php-updates.md) (CVE patches, inventory).

**Automated variant:** Atlas can build a throwaway staging copy from an encrypted backup (`<site>.staging.foxcraft.digital`, no SSH container, mail blocked). See [`wp-staging-api.md`](wp-staging-api.md). This page stays the pattern for long-lived, hand-built stagings.

---

## Folder convention

**`/opt/apps/wordpress/<subdomain>/`** — not `wp-staging`. First site: `dominikliss.foxcraft.digital`.

```
/opt/apps/wordpress/
  ├── uploads.ini            ← shared PHP limits for every WP `wp` service (see [PHP limits](#php-limits))
  └── dominikliss.foxcraft.digital/
        ├── .env                 ← chmod 600; DB_PASSWORD + DB_ROOT_PASSWORD
        ├── docker-compose.yml
        ├── Dockerfile.ssh
        ├── authorized_keys      ← copied from /opt/apps/dotnet/osgar.datahub.foxcraft.digital/authorized_keys
        └── html/                ← WordPress files, bind-mounted
```

Generate passwords with `openssl rand -hex 16`. Never commit `.env` or write token/password values into docs.

---

## Three services, one compose file

| Service | Image / build | Networks | Host ports | Notes |
|---------|---------------|----------|------------|-------|
| `wp` | `wordpress:php8.5-apache` | `edge` + `internal` | none | `container_name` = subdomain. Caddy upstream is port **80** (not 8080 like the .NET apps). Moving minor tag — patch level only via `php -v` after `docker compose pull wp`. |
| `db` | `mariadb:11` | `internal` only — never `edge` | none | Named volume `db-data`. |
| `ssh` | build `Dockerfile.ssh` | **no `networks:` key** (must not join `edge`) | `2204:22` | Named volume `ssh-keys` on `/etc/ssh` (host keys stable across rebuilds). `html/` bind-mounted to `/home/dominik/app`, `/home/pawel/app`, `/home/edison/app`. |

**PHP version must match** between the `wp` image tag and the `FROM` in `Dockerfile.ssh` (both **8.5** here). Staging should match the PHP version of production for the real site. Verified on this site: PHP 8.5.12.

---

## `docker-compose.yml`

```yaml
services:
  wp:
    image: wordpress:php8.5-apache
    container_name: dominikliss.foxcraft.digital
    environment:
      WORDPRESS_DB_HOST: db
      WORDPRESS_DB_NAME: wp
      WORDPRESS_DB_USER: wp
      WORDPRESS_DB_PASSWORD: ${DB_PASSWORD}
      WORDPRESS_CONFIG_EXTRA: |
        if (isset($$_SERVER['HTTP_X_FORWARDED_PROTO']) && $$_SERVER['HTTP_X_FORWARDED_PROTO'] === 'https') { $$_SERVER['HTTPS'] = 'on'; }
        define('WP_HOME', 'https://dominikliss.foxcraft.digital');
        define('WP_SITEURL', 'https://dominikliss.foxcraft.digital');
    volumes:
      - ./html:/var/www/html
      - ../uploads.ini:/usr/local/etc/php/conf.d/uploads.ini:ro
    depends_on:
      - db
    networks:
      - edge
      - internal
    restart: unless-stopped

  db:
    image: mariadb:11
    container_name: dominikliss.foxcraft.digital-db
    environment:
      MARIADB_DATABASE: wp
      MARIADB_USER: wp
      MARIADB_PASSWORD: ${DB_PASSWORD}
      MARIADB_ROOT_PASSWORD: ${DB_ROOT_PASSWORD}
    volumes:
      - db-data:/var/lib/mysql
    networks:
      - internal
    restart: unless-stopped

  ssh:
    build:
      context: .
      dockerfile: Dockerfile.ssh
    container_name: dominikliss.foxcraft.digital-ssh
    volumes:
      - ./html:/home/dominik/app
      - ./html:/home/pawel/app
      - ./html:/home/edison/app
      - ssh-keys:/etc/ssh
    ports:
      - "2204:22"
    restart: unless-stopped

networks:
  edge:
    external: true
  internal:

volumes:
  db-data:
  ssh-keys:
```

`$$` is Compose escaping for the PHP `$`.

New WP containers are discovered by health-api and included in the restic backup automatically (MariaDB is dumped; the raw `db-data` volume is excluded). Opt out with `health.enable=false` / `backup.exclude=true` if needed. See [`health-api.md`](health-api.md) and [`backup.md`](backup.md).

---

## PHP limits

The official `wordpress` image ships no upload limits, so the PHP default of 2 MB applies (a 7.5 MB plugin upload failed with "exceeds upload_max_filesize"). Limits are raised by mounting a php.ini fragment into `conf.d/`.

**Intended** shared file `/opt/apps/wordpress/uploads.ini` (not inside `html/` — that is the WordPress bind mount):

```ini
upload_max_filesize = 64M
post_max_size = 64M
memory_limit = 256M
max_execution_time = 300
```

Each `wp` service mounts it read-only:

```yaml
- ../uploads.ini:/usr/local/etc/php/conf.d/uploads.ini:ro
```

Live path and mount on Stella: **UNVERIFIED** (this session could not reach host SSH from the atlas.dev SSH container). Documented as the intended design. **TODO (Dominik):** confirm the file location and the compose `volumes:` line on Stella.

Changing the file after the first mount needs `docker compose up -d wp` per site. Caddy has no body size limit set — nothing to change there.

Verified 2026-10-23 on `dominikliss.foxcraft.digital`: `upload_max_filesize` and `post_max_size` are 64M inside `wp`.

**Password gotcha:** the DB password ends up in plain text in `html/wp-config.php` after first start. Later `.env` changes do **not** propagate to `wp-config.php` or to MariaDB, which only reads its passwords on first start with an empty volume.

---

## `Dockerfile.ssh`

```dockerfile
FROM php:8.5-cli

RUN apt-get update && apt-get install -y --no-install-recommends \
    openssh-server curl wget ca-certificates git procps unzip \
    && mkdir -p /var/run/sshd \
    && rm -rf /var/lib/apt/lists/*

RUN useradd -m -s /bin/bash -G www-data dominik \
    && useradd -m -s /bin/bash -G www-data pawel \
    && useradd -m -s /bin/bash -G www-data edison

RUN mkdir -p /home/dominik/.ssh /home/pawel/.ssh /home/edison/.ssh
COPY authorized_keys /home/dominik/.ssh/authorized_keys
COPY authorized_keys /home/pawel/.ssh/authorized_keys
COPY authorized_keys /home/edison/.ssh/authorized_keys

RUN chmod 700 /home/dominik/.ssh /home/pawel/.ssh /home/edison/.ssh \
    && chmod 600 /home/dominik/.ssh/authorized_keys /home/pawel/.ssh/authorized_keys /home/edison/.ssh/authorized_keys \
    && chown -R dominik:dominik /home/dominik/.ssh \
    && chown -R pawel:pawel /home/pawel/.ssh \
    && chown -R edison:edison /home/edison/.ssh

EXPOSE 22
CMD ["/usr/sbin/sshd", "-D"]
```

Notes:

- `mkdir -p /var/run/sshd` is required: `php:8.5-cli` is Debian trixie, and `openssh-server` may create the directory itself, so plain `mkdir` is fragile.
- Users are in group `www-data` (GID 33) instead of GID 1000, because WordPress files are owned by `www-data` (UID/GID 33).
- No supervisor / Node / mssql-tools (not needed for WP).
- **Deviation from [`dev-ssh-access.md`](dev-ssh-access.md):** one shared `authorized_keys` for all three users (same as osgar today), although `dev-ssh-access.md` says `edison` should have its own key file. **TODO (Dominik):** whether to split.

---

## File permissions on `html/` (run on the Stella host)

```bash
chown 33:33 html
chmod 2775 html
setfacl -R -m g:33:rwX html
setfacl -R -d -m g:33:rwX html
```

### GOTCHA — first-start ACL / SetGID wipe

After the `wp` container starts for the first time, its entrypoint copies WordPress core into `html/` with mode `644`. That lowers the ACL mask to `r--` and the SSH users get `Permission denied` on write, and the SetGID bit on `html/` was lost.

Fix, run **after first start**:

```bash
chmod -R g+rwX html
find html -type d -exec chmod g+s {} +
setfacl -R -m g:33:rwX html
setfacl -R -d -m g:33:rwX html
```

Check: `getfacl html | grep -E 'flags|mask'` shows `flags: -s-` and `mask::rwx`; then `touch ~/app/x && rm ~/app/x` inside the ssh container works.

### Residual risk (UNVERIFIED, not yet seen)

WordPress core/plugin/theme updates that `chmod` files to `0644` may lower the mask again. Possible mitigation: `FS_CHMOD_FILE` / `FS_CHMOD_DIR` in `wp-config.php`. **Not tested.** **TODO (Dominik):** decide whether to set these.

---

## Caddy block

Same template as everywhere; upstream port **80**:

```caddyfile
dominikliss.foxcraft.digital {
    tls {
        issuer acme {
            email projects@foxcraft.digital
            dir https://acme-v02.api.letsencrypt.org/directory
            dns hetzner {env.HETZNER_API_TOKEN}
        }
    }
    @blocked remote_ip 213.47.151.242 89.67.29.69 49.13.27.117
    respond @blocked 403
    reverse_proxy dominikliss.foxcraft.digital:80
}
```

DNS-01 / staging-fallback notes for this site: [`infrastructure.md`](infrastructure.md) Issues 2 and 3 (entry 2026-10-03).

---

## Order of steps for a new WP staging site

1. Folder + `.env` (`chmod 600`, passwords via `openssl rand -hex 16`).
2. `docker-compose.yml` (incl. the shared `uploads.ini` mount) + `Dockerfile.ssh` + `authorized_keys`.
3. `html/` permissions (section above).
4. Firewall rules for the new SSH port — [`infrastructure.md`](infrastructure.md) / [`dev-ssh-access.md`](dev-ssh-access.md). Next free port after this site: **2205**.
5. DNS A record; verify with `dig +short <sub> @1.1.1.1`.
6. Caddy block (upstream **80**), then `docker compose restart caddy` from `/opt/services`.
7. `docker compose up -d --build` in the app folder.
8. Re-apply the first-start permission fix (gotcha above).
9. Verify with `curl -sI https://<sub>` (expect 302 to the installer) and the issuer via `openssl s_client` (must be Let's Encrypt, **not** `(STAGING)`). Check PHP limits: `docker exec <container> php -i | grep -E 'upload_max_filesize|post_max_size'`.

---

## Client SSH config (laptop)

Host alias = subdomain:

```
Host dominikliss.foxcraft.digital
    HostName 95.217.144.93
    Port 2204
    User dominik
    IdentityFile ~/.ssh/id_ed25519
```

Verified working end to end (login, write test in `~/app`). PHP 8.5.12 CLI verified 2026-10-23.

---

## Migrating an existing live site into staging

First used for `dominikliss.foxcraft.digital` on 2026-10-11. Assumes the empty staging stack from this doc is already up.

### Source

Hetzner managed webhosting `dedi6724.your-server.de`. SSH is on port **222** and IP-protected. Stella could **not** connect to it.

### Files

Zip created on the managed host, **7.7 GB** (9.4 GB unpacked, 76,729 files). Top-level folder inside the zip is `dominikliss.com/`.

Because SSH was blocked, the zip was put into the source’s `public_html` and Stella fetched it with `wget -c` over HTTPS. The file is publicly reachable while it sits there — delete it right after the download. **UNVERIFIED** that this was done.

Unpacked to `import/extract/` (outside `html/`). `html/` was emptied (`find html -mindepth 1 -delete`) and the content moved in with `dotglob`.

Then re-applied the permission fix from this doc:

```bash
chown -R www-data:www-data html
chmod -R g+rwX html
find html -type d -exec chmod g+s {} +
setfacl -R -m g:33:rwX html
setfacl -R -d -m g:33:rwX html
```

Verified `getfacl html | grep -E 'flags|mask'` shows `flags: -s-` and `mask::rwx`.

### wp-config

The live `wp-config.php` was **not** used (it points at `localhost` and the old DB). The empty install’s config was restored. It reads `WORDPRESS_*` env vars via `getenv_docker`. Only the table prefix default was changed to `txcar_`. `WP_CACHE` is not set.

### Database

The dump was only **110 MB** (the 8 GB was the files zip). Uploaded locally via `scp` into the `-ssh` container, `docker cp` to the host, then imported into the existing `wp` DB:

```bash
docker exec -i dominikliss.foxcraft.digital-db sh -c 'mariadb -uroot -p"$MARIADB_ROOT_PASSWORD" wp' < dump.sql
```

Exit 0. 111 tables with prefix `txcar_`.

Check **before** import on MariaDB 11:

- `grep -c utf8mb4_0900 dump.sql` must be `0` (MySQL 8 collations).
- The dump must contain no `CREATE DATABASE` / `USE`.

The `wp_*` tables of the empty install stay in the DB (harmless, not dropped).

### WP-CLI

Not installed in any container. Used the phar from the official image:

```bash
docker run --rm --entrypoint cat wordpress:cli /usr/local/bin/wp > /tmp/wp-cli.phar
```

Then `docker cp` into the `wp` container and run as `www-data`.

The `-ssh` container cannot reach the DB (no network), so WP-CLI must run in the `wp` container.

Required flags: `php -d memory_limit=512M … --skip-plugins --skip-themes`. Without them the `mainwp-child` plugin hits the 128 MB limit and WP-CLI dies.

### URL replacement

Always `--dry-run` first. Always `--all-tables-with-prefix --skip-columns=guid`.

1. `https://dominikliss.com` → `https://dominikliss.foxcraft.digital` (45,712 replacements)
2. `/dominikliss.com` → `/dominikliss.foxcraft.digital` (11,742 replacements). This catches escaped (`https:\/\/`), `http`, and protocol-relative URLs but not email addresses (the leading slash excludes `@…`).

Do **not** replace the bare domain: it would rewrite email addresses (`hallo@`, `projekte@`, SMTP senders).

Leftovers intentionally kept: email addresses. The `dirsize_cache` transient (old managed-host paths) was deleted.

`siteurl` and `home` verified as the staging URL.

### GOTCHA — `.htaccess` HTTPS loop behind Caddy

The live `.htaccess` forces HTTPS with `RewriteCond %{HTTPS} !=on`. Behind Caddy, Apache only sees HTTP, which caused an endless 302 loop to itself (`curl -sIL` showed the same `Location` repeatedly).

Fix: added `RewriteCond %{HTTP:X-Forwarded-Proto} !https` directly above that line (the conditions are ANDed). After that the site returns 200. The rest of the `.htaccess` is unchanged.

### Decisions on this site

Exact copy. All plugins left active, including `mailpoet`, `wp-mail-smtp`, `mainwp-child`, `wp-rocket`, `all-in-one-wp-security-and-firewall`.

Risks: real emails can leave staging (live SMTP / MailPoet config), and `mainwp-child` carries the live site’s identity. **TODO (Dominik):** decide whether to neutralise them.

`blog_public` was set to `0`, which differs from live.

### Cleanup

Done on Stella: `import/`, the three `.bak` files, `/tmp/wp-cli.phar`.

Still open:

- the `wordpress:cli` image on the host
- the `/tmp/wp-cli.phar` inside the `wp` container (removal **not confirmed**)
- the `~/relaunch_dominikliss_com.sql` leftover in the `-ssh` container (110 MB)
- `/root/wp-before-php85.sql` on Stella (86 MB dump from the 2026-10-23 PHP 8.5 bump)
- any leftover zip / dump on the managed host (**UNVERIFIED**)

**TODO (Dominik):** clean these leftovers.
