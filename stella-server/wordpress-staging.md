# WordPress Staging Pattern

Established 2026-10-03 with `dominikliss.foxcraft.digital` as the first implementation. This is the host-wide template for WordPress staging on Stella — same role as [`dotnet-app-deployment.md`](dotnet-app-deployment.md) for .NET apps.

**Related:** [`infrastructure.md`](infrastructure.md) (Caddy, DNS, `DOCKER-USER`), [`dev-ssh-access.md`](dev-ssh-access.md) (SSH variant notes).

---

## Folder convention

**`/opt/apps/wordpress/<subdomain>/`** — not `wp-staging`. First site: `dominikliss.foxcraft.digital`.

```
/opt/apps/wordpress/
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
| `wp` | `wordpress:php8.4-apache` | `edge` + `internal` | none | `container_name` = subdomain. Caddy upstream is port **80** (not 8080 like the .NET apps). |
| `db` | `mariadb:11` | `internal` only — never `edge` | none | Named volume `db-data`. |
| `ssh` | build `Dockerfile.ssh` | **no `networks:` key** (must not join `edge`) | `2204:22` | Named volume `ssh-keys` on `/etc/ssh` (host keys stable across rebuilds). `html/` bind-mounted to `/home/dominik/app`, `/home/pawel/app`, `/home/edison/app`. |

**PHP version must match** between the `wp` image tag and the `FROM` in `Dockerfile.ssh` (both **8.4** here). Staging should match the PHP version of production for the real site. Verified on this site: PHP 8.4.26 CLI inside the SSH container.

---

## `docker-compose.yml`

```yaml
services:
  wp:
    image: wordpress:php8.4-apache
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

**Password gotcha:** the DB password ends up in plain text in `html/wp-config.php` after first start. Later `.env` changes do **not** propagate to `wp-config.php` or to MariaDB, which only reads its passwords on first start with an empty volume.

---

## `Dockerfile.ssh`

```dockerfile
FROM php:8.4-cli

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

- `mkdir -p /var/run/sshd` is required: `php:8.4-cli` is Debian trixie, and `openssh-server` may create the directory itself, so plain `mkdir` is fragile.
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
2. `docker-compose.yml` + `Dockerfile.ssh` + `authorized_keys`.
3. `html/` permissions (section above).
4. Firewall rules for the new SSH port — [`infrastructure.md`](infrastructure.md) / [`dev-ssh-access.md`](dev-ssh-access.md). Next free port after this site: **2205**.
5. DNS A record; verify with `dig +short <sub> @1.1.1.1`.
6. Caddy block (upstream **80**), then `docker compose restart caddy` from `/opt/services`.
7. `docker compose up -d --build` in the app folder.
8. Re-apply the first-start permission fix (gotcha above).
9. Verify with `curl -sI https://<sub>` (expect 302 to the installer) and the issuer via `openssl s_client` (must be Let's Encrypt, **not** `(STAGING)`).

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

Verified working end to end (login, PHP 8.4.26 CLI, write test in `~/app`).
