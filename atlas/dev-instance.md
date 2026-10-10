# Atlas dev instance (`atlas.dev.foxcraft.digital`)

Laravel Atlas running on **Stella**, not on the production CX33. Separate database, separate SSH key, separate `APP_KEY`. It can read health-api; it cannot deploy.

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

`docker-compose` next to the source. A shared `x-atlas` anchor defines the common image, `working_dir`, volumes, and network.

| Service | Image / notes | Networks |
|---------|---------------|----------|
| `app` | `php:8.5-apache` | `edge` + internal |
| `queue` | same anchor; user `33:33` | TODO: verify networks (anchor) |
| `scheduler` | same anchor; user `33:33` | TODO: verify networks (anchor) |
| `db` | `mysql:8.4`, internal only | internal |
| `ssh` | `Dockerfile.ssh`, `2205:22` | internal only — **must not** join `edge` |

Volumes: `db-data`, `ssh-keys`. Source is bind-mounted from `./src`.

### SSH container

`Dockerfile.ssh` is `php:8.5-cli` plus OpenSSH, Node 22, Composer, and the MySQL client.

Users: `dominik`, `pawel`, `edison` (group `www-data`). `authorized_keys` copied from the `dominikliss.foxcraft.digital` container.

### `src/` permissions

Owner `33:33`, `g+rwX`, SetGID on directories, ACLs for gid `33`.

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

Generated on Stella in `deploy-key/` next to `docker-compose.yml` (ed25519, comment `atlas-dev`, no passphrase, owner `33:33`, mode `600`). Mounted `:ro` into `app`, `queue`, and `scheduler` at `/var/www/.ssh/id_ed25519`.

It is **not** in any deploy target’s `authorized_keys` and **not** in deploy-api’s `allowed_signers`. It **is** a second `ddashboard` line in health-api’s own `allowed_signers` (comment `atlas-dev`). See [`../stella-server/health-api.md`](../stella-server/health-api.md).

**Consequence:** dev can read health-api and cannot call deploy-api. Deployments on this instance are untested by design. **TODO (Dominik):** test against one throwaway target later.

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

Git/GitHub works through SSH agent forwarding. The container deliberately has **no** GitHub key.

If the Cursor terminal has no `SSH_AUTH_SOCK`, run **Remote-SSH: Kill VS Code Server on Host** and reconnect.

Stella `DOCKER-USER` ACCEPTs port **2205** from Dominik’s IP, the managed host, and Edison. Next free SSH container port: **2206**. See [`../stella-server/infrastructure.md`](../stella-server/infrastructure.md).

---

## Verified

- Login
- `stella:check-health` (source health-api, 21 services)
- Scheduler
- Queue
- 0 pending migrations
