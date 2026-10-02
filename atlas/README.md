# Atlas

**Atlas** is the internal operations platform for foxcraft.digital. It is the UI and API that trigger deploys, watch Stella and WordPress-site health, and run IMAP mailbox copies.

It is **not** the business dashboard. Clients, invoices, mail, and projects live in **ddashboard**. Atlas took over the old ddashboard **Werkzeuge** screens (removed 2026-09-02).

**How Atlas sits in the company stack:** [`../integration/system-overview.md`](../integration/system-overview.md)

**Live at:** `https://dev.atlas.foxcraft.digital`  
**Host:** Hetzner managed (same account as ddashboard, `foxcrr`)  
**Repo:** `git@github.com:foxcraft/atlas.git`

---

## Stack

| Layer | What actually runs |
|-------|--------------------|
| Backend | Laravel 12, PHP 8.2, domain folders under `app/Domain/*` |
| Frontend | Vite + React 18 SPA (`resources/js`), **TanStack Router** + TanStack Query. Served from a Laravel Blade shell. Inertia packages are leftover from the Breeze starter — they are **not** the live router. |
| Auth | ddashboard tokens (`/dls/v1/auth/*`). Not Sanctum for the API. See [`auth.md`](auth.md). |
| Database | MySQL (`atlas_dev`) |
| Cache | Redis (`token:{sha256}` cache after login, general cache) |
| Deploy queue | **Database** queue `deployments`, drained by cron + `flock` (not Horizon, not a long-lived worker) |
| Scheduler | `php artisan schedule:run` every minute — Stella health + WP uptime |

---

## Modules

| Module | UI route | What it does |
|--------|----------|--------------|
| **Dashboard** | `/dashboard` | Home + Stella status summary |
| **Deployments** | `/deployments` | SSH rsync deploys and Stella deploy-api deploys |
| **WP Sites** | `/wp-sites` | Register sites, 5-min uptime, Atlas Connect auto-login |
| **Monitoring** | `/monitoring/stella` | Stella health grid + history charts |
| **Tools → IMAP Migration** | `/tools/imap-migration` | Trigger / poll `imapsync` jobs on Stella |
| **Settings** | `/settings` | SSH/GitHub key path, Brevo transactional email |

---

## Place in the system

```
Browser (IP-allowlisted)
    → Atlas SPA / API
        → ddashboard          identity (login, verify, refresh)
        → Stella deploy-api   signed .NET / Docker deploys
        → Stella health-api   signed container / TLS / disk snapshot
        → Stella imap-sync    mailbox copy (no HTTP auth)
        → GitHub + SSH hosts  ssh_rsync deploys
        → WP sites            HEAD uptime + Atlas Connect login
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
| every 5 min | `CheckWpSiteUptime` (once per site) | `HEAD https://{domain}`; `up` / `degraded` (>1 s) / `down`. Brevo email on first `down` transition. Prune logs older than 7 days. |

---

## Queue worker

Deploys only run if the worker script is invoked. Cron + `flock` prevents overlap:

```bash
/usr/home/foxcrr/public_html/dev.atlas.foxcraft.digital/scripts/queue-deployments.sh
```

It runs `php artisan queue:work database --queue=deployments --stop-when-empty` — drain pending jobs, then exit. Not a long-lived daemon. Redis in `.env` is for **cache**, not this worker.

Manual kick (stuck `pending` run):

```bash
bash /usr/home/foxcrr/public_html/dev.atlas.foxcraft.digital/scripts/queue-deployments.sh
```

Log: `/usr/home/foxcrr/logs/queue-worker.log`

---

## Authentication

All API routes except `/api/health`, `/api/auth/login`, `/api/auth/refresh`, `/api/auth/logout` require:

```
Authorization: Bearer {access_token}
```

The token is a **ddashboard** access token. Atlas proxies login to `ATLAS_DDASHBOARD_URL` and caches a successful verify. See [`auth.md`](auth.md).

Inbound HTTP is also filtered by `ATLAS_ALLOWED_IPS` (`AllowedIpMiddleware`). `/api/health` is excluded so uptime monitors can reach it.

---

## SSH key

One key, three jobs:

1. Clone private GitHub repositories
2. SSH/rsync to destination servers
3. Sign requests to Stella deploy-api and health-api (`ssh-keygen -Y sign -n deploy`)

Configured in **Settings → GitHub** (`app_settings.deploy_ssh_key_path`). Falls back to `config/deploy.php` / `DEPLOY_SSH_KEY_PATH`.

The **public** key must be in:

- GitHub (deploy key or account key) — clone
- Each destination `~/.ssh/authorized_keys` — rsync
- Stella `deploy-api/allowed_signers` as principal **`ddashboard`** — signed APIs (health-api mounts the same file)

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

### Monitoring

| Method | Path | Purpose |
|--------|------|---------|
| `GET` | `/api/monitoring/stella` | Latest Stella health check with service logs |
| `POST` | `/api/monitoring/stella/check` | On-demand health check |
| `GET` | `/api/monitoring/stella/metrics` | Recent health history (charts) |

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

---

## Required env (Atlas-specific)

See `.env.example` in the Atlas repo. The ones that wire Atlas into the rest of the system:

```
ATLAS_ALLOWED_IPS=
ATLAS_DDASHBOARD_URL=https://<ddashboard-host>
ATLAS_DEMO_MODE=false

DEPLOY_SSH_KEY_PATH=/home/atlas/.ssh/id_rsa
STELLA_DEPLOY_API_URL=https://stella-deployment-api.foxcraft.digital
STELLA_HEALTH_API_URL=https://stella-health-api.foxcraft.digital
STELLA_IMAP_SYNC_URL=https://stella.foxcraft.digital/imap-sync

# Fallback public probes (also used when health-api is down):
STELLA_API_URL=https://stella.foxcraft.digital/stella
STELLA_OLLAMA_URL=https://stella.foxcraft.digital/ollama
STELLA_CADDY_URL=https://stella.foxcraft.digital
STELLA_ADVOAPP_URL=https://advoapp.finditoo.foxcraft.digital
STELLA_OSGAR_URL=https://osgar.datahub.foxcraft.digital
```

---

## Further reading

- [`architecture.md`](architecture.md) — domains, SPA routes, hosting, demo mode
- [`auth.md`](auth.md) — ddashboard token proxy, roles, IP allowlist
- [`atlas-connect.md`](atlas-connect.md) — WP plugin contract for one-click login
- [`deploy-pipeline.md`](deploy-pipeline.md) — ssh_rsync and stella_deploy_api
- [`monitoring.md`](monitoring.md) — Stella health + WP uptime
- [`tools.md`](tools.md) — IMAP migration via Stella
- [`dotnet-azure.md`](dotnet-azure.md) — .NET / Azure via `stella_deploy_api`
- [`../integration/system-overview.md`](../integration/system-overview.md) — full company map
