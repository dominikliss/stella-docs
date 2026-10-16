# Atlas

**Atlas** is the internal operations platform for foxcraft.digital. It is the UI and API that trigger deploys, watch Stella and WordPress-site health, and run IMAP mailbox copies.

It is **not** the business dashboard. Clients, invoices, and mail live in **ddashboard**. Atlas took over the old ddashboard **Werkzeuge** screens (removed 2026-09-02). **Planned:** TrackingTime project import + task/time work in Atlas; ddashboard keeps CRM and `tracking_time_pro_id`. Spec: [`../stella-dashboard/atlas-client-lookup.md`](../stella-dashboard/atlas-client-lookup.md).

**How Atlas sits in the company stack:** [`../integration/system-overview.md`](../integration/system-overview.md)

**Live at:** `https://atlas.foxcraft.digital` (replaces `https://dev.atlas.foxcraft.digital` on the managed host)  
**Host:** Hetzner Cloud CX33 `atlas` (`2.31.42.13`) — same Cloud project as Edison, not the managed `foxcrr` host. Runbook: [`server.md`](server.md)  
**Repo:** `git@github.com:foxcraftdigital/atlas.git` (not `foxcraft/atlas`; that repo did not exist before 2026-10-10)

---

## Stack

| Layer | What actually runs |
|-------|--------------------|
| Backend | Laravel 12.61, PHP 8.5.4-FPM (pool `atlas`), domain folders under `app/Domain/*` |
| Frontend | Vite + React 18 SPA (`resources/js`), **TanStack Router** + TanStack Query. Served from a Laravel Blade shell. Inertia packages are leftover from the Breeze starter — they are **not** the live router. |
| Auth | ddashboard tokens (`/dls/v1/auth/*`). Not Sanctum for the API. See [`auth.md`](auth.md). |
| Database | MySQL 8.4 (`atlas_prod`; old managed-host name was `dev_atlas_foxcraft_digital`, not `atlas_dev`) |
| Cache | Redis localhost only (`token:{sha256}` cache after login, general cache) — **not** the deploy queue |
| Deploy queue | **Database** queue `deployments`, drained by systemd `atlas-queue.service` (not Horizon, not cron + `flock`) |
| Scheduler | `/etc/cron.d/atlas` → `php artisan schedule:run` every minute as user `atlas` |

---

## Modules

| Module | UI route | What it does |
|--------|----------|--------------|
| **Dashboard** | `/dashboard` | Home + Stella status summary |
| **Deployments** | `/deployments` | SSH rsync deploys and Stella deploy-api deploys |
| **WP Sites** | `/wp-sites` | Register sites, 5-min uptime, Atlas Connect login, start/list/explore backups |
| **Monitoring** | `/monitoring/stella`, `/monitoring/atlas` | Stella and Atlas host health grids + history charts |
| **Tools → IMAP Migration** | `/tools/imap-migration` | Trigger / poll `imapsync` jobs on Stella |
| **Tools → TrackingTime Import** | *(planned)* `/tools/tracking-time-import` | One-way import of TT projects into Atlas. Needs ddashboard [`atlas-client-lookup.md`](../stella-dashboard/atlas-client-lookup.md) first |
| **Settings** | `/settings` | SSH/GitHub key path, Brevo transactional email, Hetzner Cloud token, backup S3 + X25519 vault |

## Documents in this folder

| Document | Contents |
|----------|----------|
| [architecture.md](architecture.md) | Domains, SPA routes, hosting, queue |
| [server.md](server.md) | Production host runbook (stack, Caddy, systemd, deploy) |
| [host-api.md](host-api.md) | Atlas host API (`api.atlas.foxcraft.digital`) |
| [dev-instance.md](dev-instance.md) | Stella-hosted `atlas.dev.foxcraft.digital` |
| [auth.md](auth.md) | ddashboard token proxy, roles, IP allowlist |
| [atlas-connect/](atlas-connect/) | WP plugin: login + backups |
| [deploy-pipeline.md](deploy-pipeline.md) | ssh_rsync and stella_deploy_api |
| [monitoring.md](monitoring.md) | Stella health + WP uptime |
| [tools.md](tools.md) | IMAP copy via Stella; planned TrackingTime import |
| [dotnet-azure.md](dotnet-azure.md) | .NET / Azure via `stella_deploy_api` |

---

## Place in the system

```
Browser (IP-allowlisted)
    → Atlas SPA / API
        → ddashboard          identity (login, verify, refresh)
        → Stella deploy-api   signed .NET / Docker deploys
        → Stella health-api   signed dynamic container / Caddyfile-cert / disk / backup snapshot
        → Stella imap-sync    mailbox copy (no HTTP auth)
        → GitHub + SSH hosts  ssh_rsync deploys
        → WP sites            HEAD uptime + Atlas Connect login / backup driver
```

Atlas **does not** call Stella `/chat/*`. That is ddashboard’s AI path.

Full map: [`../integration/system-overview.md`](../integration/system-overview.md).

---

## Deployments data model

```
DeployConfig          — what to deploy (type: ssh_rsync | stella_deploy_api)
  └── DeployDestination — where (server + path + optional endpoint URLs)  [ssh_rsync only]
        └── DeployServer  — the SSH target (host, port, user)

DeployRun             — one triggered deploy (status, commit SHA / tag, message)
  └── DeployRunDestination — per-server (ssh_rsync) or single Stella job result
```

**`DeployConfig.type`** chooses the path:

- `ssh_rsync` (default) — clone from GitHub, rsync to one or more SSH servers, optional post-deploy HTTP hook
- `stella_deploy_api` — signed HTTP to Stella deploy-api; Atlas polls for completion. No SSH destinations. Stores `stella_deploy_url` and `stella_deploy_app`.

A single `ssh_rsync` config can have multiple destinations (e.g. staging then production) with a configurable cooldown between them.

---

## Architecture (runtime)

```
Browser / API caller
      │
      ▼
Laravel (IP allowlist + bearer token)
  POST /api/deploy/runs         → creates DeployRun + queues DeployRunJob
  GET  /api/deploy/runs/{id}    → poll status + per-destination logs
      │
      ▼
Database queue (deployments)
      │
      ▼
DeployRunJob (queue worker)
  If ssh_rsync:
    1. git clone --depth=1  (GitHub SSH → /tmp/atlas-deploy-run-{id}/)
    2. rsync -az --delete    (local clone → each SSH destination)
    3. POST post_deploy_endpoint (optional, per destination)
    4. cleanup tmp dir
  If stella_deploy_api:
    1. POST {stella_deploy_url}/deploy/{app}  (signed payload)
    2. poll GET /deploy/{app}/status/{job_id} every 5s (up to 30 min)
    3. store commit SHA + message from Stella response
```

Code layout: [`architecture.md`](architecture.md).

---

## Scheduled jobs

Both health loops run via `php artisan schedule:run` (cron every minute). They use `dispatchSync` — they do **not** go through the deployments queue.

| Schedule | Job | Effect |
|----------|-----|--------|
| every 5 min | `CheckStellaHealth` | Signed health-api request; store `StellaHealthCheck` + per-service `StellaServiceUptimeLog`. Public-endpoint probes if health-api is down. Prune older than 30 days. |
| every 5 min | `CheckAtlasHealth` | Signed host-api request; store `AtlasHealthCheck` + per-service `AtlasServiceUptimeLog`. Prune older than 30 days. |
| every 5 min | `CheckWpSiteUptime` (once per site) | `HEAD https://{domain}`; `up` / `degraded` (>1 s) / `down`. Brevo email on first `down` transition. Prune logs older than 7 days. |
| every minute | `wp-sites:drive-backups` | Backups feature still in development; backup driver not finished |

---

## Queue worker

On production (`atlas`), deploys run via systemd **`atlas-queue.service`** (User=`atlas`, `Restart=always`):

```bash
php artisan queue:work database --queue=deployments --sleep=3 --timeout=7200 --max-time=3500
```

Logs: `journalctl -u atlas-queue`. Redis is **cache only**, not this worker.

This replaces cron + `flock` `scripts/queue-deployments.sh`. That script and its lock/log paths under `/usr/home/foxcrr/` no longer apply on `atlas`. See [`server.md`](server.md).

The old managed-host instance still has those two crons until they are deactivated. **TODO (Dominik):** confirm deactivated and decide when to retire `dev.atlas.foxcraft.digital`.

---

## Authentication

All API routes except `/api/health`, `/api/auth/login`, `/api/auth/refresh`, `/api/auth/logout` require:

```
Authorization: Bearer {access_token}
```

The token is a **ddashboard** access token. Atlas proxies login to `ATLAS_DDASHBOARD_URL` and caches a successful verify. See [`auth.md`](auth.md).

Inbound HTTP is also filtered by `ATLAS_ALLOWED_IPS` (`AllowedIpMiddleware`, prepended globally in `bootstrap/app.php`). `withoutMiddleware()` on `GET /api/health` never took effect; the route returns early inside the middleware (`$request->is('api/health')`). Fixed and deployed 2026-10-10. **TODO (Dominik):** commit hash / tag. See [`auth.md`](auth.md).

---

## SSH key

One key, three jobs:

1. Clone private GitHub repositories
2. SSH/rsync to destination servers
3. Sign requests to Stella deploy-api and health-api (`ssh-keygen -Y sign -n deploy`)

Configured in **Settings → GitHub** (`app_settings.deploy_ssh_key_path` = `/home/atlas/.ssh/id_ed25519` on production; the DB value overrides `.env`). Falls back to `config/deploy.php` / `DEPLOY_SSH_KEY_PATH`.

Production reuses the same keypair as the old managed-host Atlas (principal `ddashboard`, key comment `atlas-prod`) so the 28 rows in `deploy_servers` did not need new `authorized_keys`. The managed host still holds that key and can still deploy/sign. Rotation was considered and deferred.

The **public** key must be in:

- GitHub (deploy key or account key) — clone
- Each destination `~/.ssh/authorized_keys` — rsync
- Stella `deploy-api/allowed_signers` as principal **`ddashboard`** — deploy-api (production key only)
- Stella `health-api/allowed_signers` as principal **`ddashboard`** — health-api’s **own** file (production `atlas-prod` plus dev `atlas-dev`). See [`../stella-server/health-api.md`](../stella-server/health-api.md).

---

## Key API endpoints

### Deployments

| Method | Path | Purpose |
|--------|------|---------|
| `GET` | `/api/deploy/configs` | List all deploy configurations |
| `POST` | `/api/deploy/configs` | Create a deploy config |
| `PATCH` | `/api/deploy/configs/{id}` | Update a deploy config |
| `DELETE` | `/api/deploy/configs/{id}` | Delete a deploy config |
| `POST` | `/api/deploy/configs/{id}/test` | Test SSH connectivity for all destinations |
| `POST` | `/api/deploy/configs/{id}/clear-page-cache` | Call the clear-page-cache endpoint |
| `POST` | `/api/deploy/configs/{id}/regenerate-unused-css` | Call the regenerate-CSS endpoint |
| `GET` | `/api/deploy/servers` | List SSH servers |
| `POST` | `/api/deploy/servers` | Create an SSH server |
| `PATCH` | `/api/deploy/servers/{id}` | Update an SSH server |
| `DELETE` | `/api/deploy/servers/{id}` | Delete an SSH server |
| `POST` | `/api/deploy/servers/{id}/test` | Test SSH connection |
| `POST` | `/api/deploy/runs` | Trigger a deploy run |
| `GET` | `/api/deploy/runs` | List deploy runs |
| `GET` | `/api/deploy/runs/{id}` | Poll run status + logs |
| `GET` | `/api/deploy/active-run` | Any currently-running runs |
| `POST` | `/api/deploy/runs/{id}/cancel` | Cancel a running deploy |
| `GET` | `/api/deploy/stella/apps` | List apps on Stella deploy-api (signed) |
| `POST` | `/api/deploy/ssh-browse` | Browse remote directory over SSH |
| `GET` | `/api/deploy/github/branches` | List GitHub branches for a repo |

### WP Sites

| Method | Path | Purpose |
|--------|------|---------|
| `GET` | `/api/wp-sites` | List all WP sites with latest uptime status |
| `POST` | `/api/wp-sites` | Register a new WP site |
| `PATCH` | `/api/wp-sites/{id}` | Update a WP site |
| `DELETE` | `/api/wp-sites/{id}` | Remove a WP site |
| `POST` | `/api/wp-sites/{id}/login` | One-time WP admin URL via Atlas Connect |
| `POST` | `/api/wp-sites/{id}/refresh` | Immediate uptime check |
| `GET` | `/api/wp-sites/{id}/backups` | List site backups |
| `POST` | `/api/wp-sites/{id}/backups` | Start a backup (`full` / `db` / `files`) |
| `GET` | `/api/wp-sites/{id}/backups/{backup}` | Show one backup |
| `GET` | `/api/wp-sites/{id}/backups/{backup}/tree` | List objects in the backup prefix |
| `POST` | `/api/wp-sites/{id}/backups/{backup}/objects` | Decrypt one backup object |
| `POST` | `/api/wp-sites/{id}/backups/{backup}/cancel` | Cancel a running backup |

### Monitoring

| Method | Path | Purpose |
|--------|------|---------|
| `GET` | `/api/monitoring/stella` | Latest Stella health check with service logs |
| `POST` | `/api/monitoring/stella/check` | On-demand health check |
| `GET` | `/api/monitoring/stella/metrics` | Recent health history (charts) |
| `GET` | `/api/monitoring/atlas` | Latest Atlas host health check with service logs |
| `POST` | `/api/monitoring/atlas/check` | On-demand Atlas host check |
| `GET` | `/api/monitoring/atlas/metrics` | Recent Atlas host history (charts) |
| `POST` | `/api/monitoring/atlas/deploy` | Start a production deploy (atlas.dev; 404 when `ATLAS_SELF_DEPLOY_ENABLED` is off) |
| `GET` | `/api/monitoring/atlas/deploy/status` | Poll production deploy status (atlas.dev; 404 when off) |

### Tools

| Method | Path | Purpose |
|--------|------|---------|
| `POST` | `/api/tools/imap-migration/test-connection` | Test IMAP credentials before starting |
| `GET` | `/api/tools/imap-migration/jobs` | List Atlas-side job history |
| `POST` | `/api/tools/imap-migration/start` | Start an imapsync job on Stella |
| `GET` | `/api/tools/imap-migration/jobs/{id}/status` | Poll + sync status from Stella |
| `GET` | `/api/tools/imap-migration/jobs/{id}/log` | Fetch job log from Stella |
| `DELETE` | `/api/tools/imap-migration/jobs/{id}` | Cancel a running job |

### Settings

| Method | Path | Purpose |
|--------|------|---------|
| `GET` | `/api/settings/github` | Show GitHub/SSH key settings |
| `PATCH` | `/api/settings/github` | Update GitHub/SSH key settings |
| `POST` | `/api/settings/github/test` | Test SSH key against GitHub |
| `GET` | `/api/settings/email` | Show Brevo email settings |
| `PATCH` | `/api/settings/email` | Update Brevo settings |
| `POST` | `/api/settings/email/test` | Send a test email via Brevo |
| `GET` | `/api/settings/backups` | Show backup vault (S3 + public key; private key hidden) |
| `PATCH` | `/api/settings/backups` | Update backup S3 / keys |
| `POST` | `/api/settings/backups/generate-key` | Generate an X25519 key pair |
| `GET` | `/api/settings/hetzner` | Show Hetzner Cloud token status (never the token) |
| `PATCH` | `/api/settings/hetzner` | Save the read-only Cloud token (encrypted) |
| `POST` | `/api/settings/hetzner/test` | List servers in the project |

---

## Required env (Atlas-specific)

See `.env.example` in the Atlas repo. The ones that wire Atlas into the rest of the system. Production `ATLAS_ALLOWED_IPS` is `194.126.177.181`, `23.88.90.12` (managed host), `95.217.144.93` (Stella). **TODO (Dominik):** remove `23.88.90.12` unless something on the managed host really calls Atlas; reason for each entry unverified.

```
ATLAS_ALLOWED_IPS=194.126.177.181,23.88.90.12,95.217.144.93
ATLAS_DDASHBOARD_URL=https://data3.dominikliss.com/wp-json
ATLAS_DEMO_MODE=false

DEPLOY_SSH_KEY_PATH=/home/atlas/.ssh/id_ed25519
STELLA_DEPLOY_API_URL=https://stella-deployment-api.foxcraft.digital
STELLA_HEALTH_API_URL=https://stella-health-api.foxcraft.digital
ATLAS_HOST_API_URL=https://api.atlas.foxcraft.digital
ATLAS_SELF_DEPLOY_ENABLED=false
STELLA_IMAP_SYNC_URL=https://stella.foxcraft.digital/imap-sync

# Fallback public probes (also used when health-api is down):
STELLA_API_URL=https://stella.foxcraft.digital/stella
STELLA_OLLAMA_URL=https://stella.foxcraft.digital/ollama
STELLA_CADDY_URL=https://stella.foxcraft.digital
STELLA_ADVOAPP_URL=https://advoapp.finditoo.foxcraft.digital
STELLA_OSGAR_URL=https://osgar.datahub.foxcraft.digital

BACKUP_S3_ACCESS_KEY=
BACKUP_S3_SECRET_KEY=
```

---

## Further reading

- [`server.md`](server.md) — production host runbook
- [`host-api.md`](host-api.md) — Atlas host API (`api.atlas.foxcraft.digital`)
- [`dev-instance.md`](dev-instance.md) — Stella-hosted `atlas.dev`
- [`architecture.md`](architecture.md) — domains, SPA routes, hosting, demo mode
- [`auth.md`](auth.md) — ddashboard token proxy, roles, IP allowlist
- [`atlas-connect/`](atlas-connect/) — WP plugin: login + backups (Atlas driver + Settings vault + WP Sites UI)
- [`deploy-pipeline.md`](deploy-pipeline.md) — ssh_rsync and stella_deploy_api
- [`monitoring.md`](monitoring.md) — Stella health, Atlas host health, WP uptime
- [`tools.md`](tools.md) — IMAP migration via Stella
- [`dotnet-azure.md`](dotnet-azure.md) — .NET / Azure via `stella_deploy_api`
- [`../integration/system-overview.md`](../integration/system-overview.md) — full company map
