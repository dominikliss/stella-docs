# Atlas dev instance (`atlas.dev.foxcraft.digital`)

Laravel Atlas running on **Stella**, not on the production CX33. Separate database, separate SSH key, separate `APP_KEY`. It can read health-api. It cannot call Stella deploy-api. A customer deployment from this instance succeeded on 2026-10-11 (see [Dev SSH key](#dev-ssh-key)).

**Production runbook:** [`server.md`](server.md)  
**Product / API:** [`README.md`](README.md)

---

## Hosting

| | |
|---|---|
| **URL** | `https://atlas.dev.foxcraft.digital` |
| **Host** | Stella (`95.217.144.93`) |
| **Path** | `/opt/apps/laravel/atlas.dev.foxcraft.digital/` |
| **DNS** | A record `atlas.dev` → `95.217.144.93` (zone `567656`) |
| **TLS** | Let's Encrypt production via DNS-01 |
| **Caddy** | Block in `/opt/services/caddy/Caddyfile`: 403 for everything except `194.126.177.181`, `reverse_proxy` to `atlas.dev.foxcraft.digital:80` |

---

## Compose

`docker-compose` next to the source. A shared `x-atlas` anchor defines the common image, `working_dir`, volumes, and network. The anchor mounts three files from `./deploy-key/` read-only into `app`, `queue`, and `scheduler`: `id_ed25519`, `id_ed25519.pub`, and `known_hosts` (see [Dev SSH key](#dev-ssh-key)).

| Service | Image / notes | Networks |
|---------|---------------|----------|
| `app` | Locally built `atlasdev-app` (`build: .`; Dockerfile `FROM php:8.5-apache`; `x-atlas` sets `image: atlasdev-app`). Not a pulled image — `docker compose pull app` fails with "pull access denied". | `edge` + internal |
| `queue` | same image via `x-atlas`; user `33:33` | TODO: verify networks (anchor) |
| `scheduler` | same image via `x-atlas`; user `33:33` | TODO: verify networks (anchor) |
| `db` | `mysql:8.4`, internal only | internal |
| `ssh` | `Dockerfile.ssh`, `2205:22` | internal only — **must not** join `edge` |

Volumes: `db-data`, `ssh-keys`. Source is bind-mounted from `./src`.

### SSH container

`Dockerfile.ssh` is `php:8.5-cli` plus OpenSSH, Node 22, Composer, and the MySQL client.

Users: `dominik`, `pawel`, `edison` (group `www-data`). `authorized_keys` copied from the `dominikliss.foxcraft.digital` container.

### `src/` permissions

Owner `33:33`, `g+rwX`, SetGID on directories, ACLs for gid `33`.

### Updating PHP

`app` / `queue` / `scheduler` share `atlasdev-app`. Check no customer deploy is running before recreating `queue`. `db` is untouched. Full runbook: [`../stella-server/php-updates.md`](../stella-server/php-updates.md).

```bash
docker compose build --pull app
docker compose up -d app queue scheduler
docker compose build --pull ssh
docker compose up -d ssh
```

Verified 2026-10-23: PHP 8.5.12 in `app`; all five containers Up (`atlas.dev.foxcraft.digital`, `-db`, `-queue`, `-scheduler`, `-ssh`); Caddy returns 403 from Stella itself (expected, IP filter). SSH container PHP version after the rebuild: **UNVERIFIED**.

---

## `.env` (dev)

Do not write secret values here. This instance has its **own** `APP_KEY` (not copied from production).

| Key | Dev |
|-----|-----|
| `APP_ENV` | `local` |
| `APP_DEBUG` | `true` |
| `DB_DATABASE` | `atlas_dev` |
| `CACHE_STORE` | `file` |
| `QUEUE_CONNECTION` | `database` |
| `BROADCAST_CONNECTION` | `log` (no Redis / Reverb container) |
| `ATLAS_ALLOWED_IPS` | empty — Caddy does the IP filtering |
| `DEPLOY_SSH_KEY_PATH` | `/var/www/.ssh/id_ed25519` |

---

## Dev SSH key

Generated on Stella in `deploy-key/` next to `docker-compose.yml` (ed25519, comment `atlas-dev`, no passphrase, owner `33:33`, mode `600`). The `x-atlas` anchor mounts three files from `./deploy-key/` read-only into `app`, `queue`, and `scheduler`:

| Host file | Container path |
|-----------|----------------|
| `id_ed25519` | `/var/www/.ssh/id_ed25519` |
| `id_ed25519.pub` | `/var/www/.ssh/id_ed25519.pub` |
| `known_hosts` | `/var/www/.ssh/known_hosts` |

Reasons for the extra mounts (2026-10-11):

- The Atlas GitHub settings page reported “public key missing” when only the private key was mounted.
- `known_hosts` was created with `ssh-keyscan -t ed25519 github.com > deploy-key/known_hosts` (owner `33:33`, mode `644`). It is writable for `www-data`.
- `/var/www/.ssh/` itself is root-owned (Docker created it), so SSH still logs a harmless `hostfile_replace_entries: mkstemp: Permission denied` warning (`UpdateHostKeys`; same class of problem as the Edison note in [`../edison/cursor-agent.md`](../edison/cursor-agent.md)). It does not break clones.

It is documented as **not** in any deploy target’s `authorized_keys` and **not** in deploy-api’s `allowed_signers`. It **is** a second `ddashboard` line in health-api’s own `allowed_signers` (comment `atlas-dev`). See [`../stella-server/health-api.md`](../stella-server/health-api.md).

It is registered as an account-level SSH key on the GitHub account `dominikliss` (GitHub answers `Hi dominikliss!`), which can read the org repo `foxcraftdigital/dominikliss`. **TODO (Dominik):** consider a read-only deploy key instead, since an account key reaches every repo that account can read.

**Consequence:** dev can read health-api. It still cannot call Stella deploy-api (key not in deploy-api’s `allowed_signers`). A customer deployment from this instance **did** succeed on 2026-10-11, so deployments are no longer “untested by design”:

- Target: `liss01@dedi6724.your-server.de`, path `/usr/home/liss01/public_html/dominikliss.com/wp-content/themes/dominikliss`
- Source: `foxcraftdigital/dominikliss`, branch `main`, commit `4ebbe05`, via rsync
- Log ended with “Deployment successful”. The rsync printed `cannot delete non-empty directory: node_modules/…` messages (harmless).
- The target was the **live** site, which Dominik confirmed was intended.

**UNVERIFIED:** why the `atlas-dev` key could reach that host, given this page still records it as in no `authorized_keys`. **TODO (Dominik):** check which `deploy_servers` entry and key were used.

---

## Role seed

A fresh database after `migrate` needs roles `admin`, `developer`, `pm`, `viewer` with guard `web`.

`php artisan atlas:setup` cannot be used here — its config check pings Redis and aborts (dev has no Redis).

Seed via tinker. Use `-e HOME=/tmp` (otherwise psysh cannot write its config). **TODO: verify** the exact `docker exec` / tinker invocation used to create the four roles.

---

## Cursor / SSH

```
Host atlas.dev.foxcraft.digital
    HostName 95.217.144.93
    Port 2205
    User dominik
    ForwardAgent yes
```

Git/GitHub in the **`-ssh` container** works through SSH agent forwarding. That container deliberately has **no** GitHub key. The Atlas `app` / `queue` / `scheduler` containers use the `atlas-dev` key for GitHub (see [Dev SSH key](#dev-ssh-key)).

If the Cursor terminal has no `SSH_AUTH_SOCK`, run **Remote-SSH: Kill VS Code Server on Host** and reconnect.

Stella `DOCKER-USER` ACCEPTs port **2205** from Dominik’s IP, the managed host, and Edison. Next free SSH container port: **2208** (2206 hypershape, 2207 Edison dev). See [`../stella-server/infrastructure.md`](../stella-server/infrastructure.md).

### GitHub clone gotchas (2026-10-11)

- `SshDeployService::cloneLocally` builds `git@github.com:{repo}.git`. The repo field in the deploy form must be `owner/name` **without** `.git`. With `.git` the URL becomes `….git.git` and GitHub answers “Repository not found”, which looks like a key problem but is not.
- The settings field “GitHub Username” is not used in the clone URL. It can stay empty.
- The settings page shows “CONNECTED” after “Test Connection”.

---

## Verified

- Login
- `stella:check-health` (source health-api, 21 services at first check). Later 2026-10-10, after health-api became dynamic: 20 containers + 7 certs. See [`../stella-server/health-api.md`](../stella-server/health-api.md).
- Scheduler
- Queue
- 0 pending migrations
