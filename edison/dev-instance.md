# Edison dev instance (`edison.dev.foxcraft.digital`)

Laravel Edison app running on **Stella**, not on the Edison server. Same pattern as [`../atlas/dev-instance.md`](../atlas/dev-instance.md). Separate database, separate `APP_KEY`. The agent backend is stubbed: there is no link to the real `cursor-agent` on Edison yet.

**App / repo:** [`app.md`](app.md)  
**Edison host:** [`README.md`](README.md)

Set up 2026-10-26. Naming scheme for dev instances is `<tool>.dev` (no hyphens).

---

## Hosting

| | |
|---|---|
| **URL** | `https://edison.dev.foxcraft.digital` |
| **Host** | Stella (`95.217.144.93`) |
| **Path** | `/opt/apps/laravel/edison.dev.foxcraft.digital/` |
| **DNS** | A record `edison.dev` → `95.217.144.93`, TTL 300 (zone `567656`), created via the Hetzner Cloud API. Verified with `dig @8.8.8.8`. |
| **TLS** | Let's Encrypt production via DNS-01. Issued in about 27 s on the first attempt; no `propagation_delay` needed. |
| **Caddy** | Block at the bottom of `/opt/services/caddy/Caddyfile` with the `dir` pin: `@blocked not remote_ip 194.126.177.181` (403 for everything else), `reverse_proxy edison.dev.foxcraft.digital:80` |
| **SSH** | Port **2207** (`ssh` service) |

### Caddy change (2026-10-26)

The Caddyfile was replaced via `rm` + nano (backup `/root/Caddyfile.bak`), then:

```bash
docker compose restart caddy
docker compose up -d --force-recreate health-api
```

`health-api` needs the force-recreate because it bind-mounts the Caddyfile as a single file (Issue 4 in [`../stella-server/infrastructure.md`](../stella-server/infrastructure.md)).

### Verified (2026-10-26)

- HTTPS returns 200 from Apache / PHP 8.5.12 via Caddy from the whitelisted IP.
- From Stella itself, Caddy returns 403. Expected: Stella's IP is not on the allowlist.
- To test Apache directly, bypassing Caddy:

```bash
docker exec edison.dev.foxcraft.digital curl -sI http://localhost
```

---

## Compose

Modeled on atlas.dev, **without** the `deploy-key` mounts and **without** the `deployments` queue name. A shared `x-edison` anchor defines `image: edisondev-app`, `working_dir: /var/www/html`, the `./src` bind mount, `depends_on: db`, and the internal network.

| Service | Image / notes | Networks |
|---------|---------------|----------|
| `app` | Locally built `edisondev-app` (`build: .`), container `edison.dev.foxcraft.digital` | `edge` + internal |
| `queue` | same image via `x-edison`; user `33:33`; `php artisan queue:work database --sleep=3 --timeout=7200 --max-time=3500` | internal (anchor) |
| `scheduler` | same image via `x-edison`; user `33:33`; `php artisan schedule:work` | internal (anchor) |
| `db` | `mysql:8.4`, database `edison_dev`, user `edison_dev`, volume `db-data` | internal |
| `ssh` | `Dockerfile.ssh`, `2207:22`; `./src` mounted for `dominik`, `pawel`, `edison`; volume `ssh-keys` | internal only — **must not** join `edge` |

The queue worker exits after about an hour (`--max-time=3500`) by design; Docker restarts it.

All five containers (`app`, `db`, `ssh`, `queue`, `scheduler`) were up on 2026-10-26. health-api and the restic backup discover new containers by default; **UNVERIFIED** for these containers.

### Folder contents

```
/opt/apps/laravel/edison.dev.foxcraft.digital/
  ├── .env                 ← compose secrets, mode 600 (DB_PASSWORD, DB_ROOT_PASSWORD)
  ├── Dockerfile
  ├── Dockerfile.ssh
  ├── authorized_keys
  ├── docker-compose.yml
  └── src/                 ← Edison repo checkout (Laravel .env lives here)
```

`DB_PASSWORD` and `DB_ROOT_PASSWORD` were generated with `openssl rand -hex 16`. Values are not recorded here.

### Dockerfile

Same as atlas.dev, without `rsync` and `openssh-client` (no deploy key, no deployments). The Apache conf is named `edison.conf`.

### SSH container

`Dockerfile.ssh` is copied unchanged from atlas.dev: `php:8.5-cli` plus OpenSSH, Node 22, Composer, and the MySQL client. Tools seen in the image: PHP 8.5.12 CLI, Composer 2.10.3, Node 22, git, MySQL client.

- `git safe.directory` for all three users is baked into the image. This resolves the open TODO for new SSH containers; osgar is still open (see [`../stella-server/dev-ssh-access.md`](../stella-server/dev-ssh-access.md)).
- No `nano` or other editor in the image. Use Cursor Remote-SSH, or edit on the Stella host (`src/` is the same bind mount).
- `authorized_keys` is copied from atlas.dev and shared by `dominik`, `pawel`, and `edison`. Same open TODO as osgar and dominikliss: **TODO (Dominik):** whether to split it.
- The `ssh` service has no home volume. Git identity, `known_hosts` for github.com, and clones in the home directory (e.g. `~/atlas-ref`) were set up by hand and are **lost on a rebuild**. **TODO (Dominik):** decide whether to add a home volume.

### `src/` permissions

Owner `www-data`, mode `2775` (SetGID), ACL `g:33:rwX` plus a default ACL.

---

## `.env` (dev)

Do not write secret values here. This instance has its **own** `APP_KEY` and a fresh database (nothing copied from Atlas). `src/.env` is the same file as `~/app/.env` inside the `ssh` container.

| Key | Dev |
|-----|-----|
| `APP_ENV` | `local` |
| `APP_DEBUG` | `true` |
| `APP_URL` | `https://edison.dev.foxcraft.digital` |
| `DB_CONNECTION` / `DB_HOST` | `mysql` / `db` |
| `DB_DATABASE` | `edison_dev` |
| `DB_PASSWORD` | filled from the compose folder's `.env` without printing it |
| `SESSION_DRIVER` | `database` |
| `CACHE_STORE` | `file` |
| `QUEUE_CONNECTION` | `database` |
| `BROADCAST_CONNECTION` | `log` |
| `EDISON_ALLOWED_IPS` | empty — Caddy does the IP filtering |
| `EDISON_DDASHBOARD_URL` | set (required, see [Incident](#incident-2026-10-26--empty-edison_ddashboard_url)) |
| `EDISON_DEMO_MODE` | `false` |

---

## Role seed

Done 2026-10-26: `php artisan migrate` (teams + users update, permission tables, `prepare_users_for_ddashboard_auth`) and `php artisan db:seed` (`RolesSeeder`: `admin`, `developer`, `pm`, `viewer`, guard `web`). A later `php artisan migrate` reported nothing to migrate.

---

## Incident 2026-10-26 — empty `EDISON_DDASHBOARD_URL`

With `EDISON_DDASHBOARD_URL` empty, the login call failed with `Illuminate\Http\Client\ConnectionException` "URI must include a scheme and host". In `storage/logs/laravel.log`:

- `17:52:27`–`17:52:28` (four entries): API checks during setup against a local `php artisan serve` inside the container; the override was not passed to the server process.
- `18:05:50`: login through `https://edison.dev.foxcraft.digital`.

Resolved by setting the variable. **TODO (Dominik):** make the driver throw a clear "EDISON_DDASHBOARD_URL is not set" error.

---

## Cursor / SSH

```
Host edison.dev.foxcraft.digital
    HostName 95.217.144.93
    Port 2207
    User dominik
    ForwardAgent yes
```

Git/GitHub in the `-ssh` container works through SSH agent forwarding; GitHub keys are never stored in the container. Verified: `ssh -T git@github.com` answers `Hi dominikliss!`. Do not set `ForwardAgent` under `Host *`.

Stella `DOCKER-USER` ACCEPTs port **2207** from Dominik's IP (`194.126.177.181`), the managed host (`23.88.90.12`), and Edison (`edison-ip` `178.105.203.54`), then DROP. Verified live in `iptables`. Next free SSH container port: **2208**. See [`../stella-server/infrastructure.md`](../stella-server/infrastructure.md).

---

## Verified

- HTTPS via Caddy (200 from the whitelisted IP, 403 from Stella)
- All five containers up
- SSH on 2207 and GitHub via the forwarded agent
- 0 pending migrations; roles seeded

**UNVERIFIED:**

- End-to-end login through the real ddashboard. The auth endpoints were only exercised against a local mock ddashboard during setup.
- Whether ddashboard's IP allowlist accepts Stella (`95.217.144.93`) for `/dls/v1/auth/*`. Stella previously got 403 from the managed host's Apache (see [`../open-gaps.md`](../open-gaps.md)). The allowlist is probably an `.htaccess` under `public_html` on the ddashboard host.
- health-api / backup discovery of the five containers.
