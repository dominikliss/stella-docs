# Atlas

**Atlas** is the internal operations platform for foxcraft.digital projects. It provides a web UI and API to orchestrate deployments, monitor service health, track WP site uptime, and manage IMAP mail migrations.

**Live at:** `https://dev.atlas.foxcraft.digital`  
**Stack:** Laravel 12, React + Inertia.js, MySQL, database-backed queue  
**Repo:** `git@github.com:foxcraft/atlas.git`

---

## Modules

| Module | What it does |
|---|---|
| **Deployments** | Trigger SSH rsync deploys or Stella Deployment API deploys from GitHub to production servers |
| **WP Sites** | Track registered WordPress sites; ping uptime every 5 minutes; email alert on down transition |
| **Monitoring** | Poll Stella health-api every 5 minutes; store per-service history; display status grid |
| **Tools → IMAP Migration** | UI for triggering and monitoring imapsync jobs via the Stella `imap-sync` service |
| **Settings** | Manage SSH/GitHub key and Brevo transactional email credentials |

---

## Deployments data model

```
DeployConfig          — what to deploy (type: ssh_rsync | stella_deploy_api)
  └── DeployDestination — where (server + path + optional endpoint URLs)  [ssh_rsync only]
        └── DeployServer  — the SSH target (host, port, user)

DeployRun             — one triggered deploy (status, commit SHA, log per destination)
  └── DeployRunDestination — per-server (ssh_rsync) or single Stella job result
```

**`DeployConfig.type`** determines the deploy path:
- `ssh_rsync` (default) — clone from GitHub, rsync to one or more SSH servers, optional post-deploy HTTP hook
- `stella_deploy_api` — call the Stella deploy-api via signed HTTP; Atlas polls for completion. No SSH destinations. Stores `stella_deploy_url` and `stella_deploy_app`.

A single `ssh_rsync` config can have multiple `DeployDestination`s — e.g. deploy the same repo to staging and production in one run, with a configurable cooldown delay between destinations.

---

## Architecture

```
Browser / API caller
      │
      ▼
Laravel (token auth)
  POST /api/deploy/runs         → creates DeployRun + queues DeployRunJob
  GET  /api/deploy/runs/{id}    → poll status + per-destination logs
      │
      ▼
Database queue (deployments queue)
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

---

## Scheduled jobs

Both cron-driven health loops run via `php artisan schedule:run` (called by cron every minute, standard Laravel scheduler):

| Schedule | Job | Effect |
|---|---|---|
| every 5 min | `CheckStellaHealth` | Signs a health-api request, stores a `StellaHealthCheck` + per-service `StellaServiceUptimeLog` rows. Falls back to direct public endpoint probes if health-api is unreachable. Prunes records older than 30 days. |
| every 5 min | `CheckWpSiteUptime` (once per site) | Curl HEAD to `https://{domain}`; marks `up` / `degraded` (>1 s) / `down`. Sends a Brevo email alert on first `down` transition. Prunes logs older than 7 days. |

These run synchronously within the scheduler call (`dispatchSync`), not via the deployments queue.

---

## Queue worker

The queue must be running for deploys to execute. The worker is driven by a cron-invoked script that uses `flock` to prevent overlapping processes:

```bash
/usr/home/foxcrr/public_html/dev.atlas.foxcraft.digital/scripts/queue-deployments.sh
```

It runs `php artisan queue:work database --queue=deployments --stop-when-empty` — meaning it processes all pending jobs then exits. This is invoked by cron rather than running as a long-lived daemon.

To trigger it manually (e.g. when a deploy is stuck pending):
```bash
bash /usr/home/foxcrr/public_html/dev.atlas.foxcraft.digital/scripts/queue-deployments.sh
```

Log output: `/usr/home/foxcrr/logs/queue-worker.log`

---

## Authentication

All API routes require a **bearer token** (stored in `personal_access_tokens` via Sanctum or managed via the ddashboard token driver):
```
Authorization: Bearer {token}
```

Tokens are managed in the Atlas UI (Settings → API tokens) or via `php artisan sanctum:prune-expired`.

---

## SSH key

Atlas uses a **single SSH key** for:
1. Cloning private GitHub repositories
2. SSH-ing to destination servers for rsync (ssh_rsync deploys)
3. Signing requests to Stella's deploy-api and health-api (SSH-signature auth)

Configured in: **Settings → GitHub** (stored in `app_settings` table under key `deploy_ssh_key_path`). Falls back to `config/deploy.php` → `~/.ssh/id_rsa`.

The key's **public key** must be added to:
- GitHub (deploy key or personal account key) — for cloning
- Each destination server's `~/.ssh/authorized_keys` — for rsync
- Stella's `deploy-api/allowed_signers` (as the `ddashboard` principal) — for signed API calls

---

## Key API endpoints

### Deployments

| Method | Path | Purpose |
|---|---|---|
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
| `GET` | `/api/deploy/stella/apps` | List apps available on Stella deploy-api |
| `POST` | `/api/deploy/ssh-browse` | Browse remote directory over SSH |
| `GET` | `/api/deploy/github/branches` | List GitHub branches for a repo |

### WP Sites

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/api/wp-sites` | List all WP sites with latest uptime status |
| `POST` | `/api/wp-sites` | Register a new WP site |
| `PATCH` | `/api/wp-sites/{id}` | Update a WP site |
| `DELETE` | `/api/wp-sites/{id}` | Remove a WP site |
| `POST` | `/api/wp-sites/{id}/login` | Generate a WP auto-login URL via secret key |
| `POST` | `/api/wp-sites/{id}/refresh` | Trigger an immediate uptime check |

### Monitoring

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/api/monitoring/stella` | Latest Stella health check with service logs |
| `POST` | `/api/monitoring/stella/check` | Run an on-demand health check |
| `GET` | `/api/monitoring/stella/metrics` | Recent health history (for charts) |

### Tools

| Method | Path | Purpose |
|---|---|---|
| `POST` | `/api/tools/imap-migration/test-connection` | Test IMAP credentials before starting a job |
| `GET` | `/api/tools/imap-migration/jobs` | List all migration jobs |
| `POST` | `/api/tools/imap-migration/start` | Start an imapsync job via Stella |
| `GET` | `/api/tools/imap-migration/jobs/{id}/status` | Poll job status |
| `GET` | `/api/tools/imap-migration/jobs/{id}/log` | Fetch job log |
| `DELETE` | `/api/tools/imap-migration/jobs/{id}` | Cancel a running job |

### Settings

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/api/settings/github` | Show GitHub/SSH key settings |
| `PATCH` | `/api/settings/github` | Update GitHub/SSH key settings |
| `POST` | `/api/settings/github/test` | Test SSH key against GitHub |
| `GET` | `/api/settings/email` | Show Brevo email settings |
| `PATCH` | `/api/settings/email` | Update Brevo settings |
| `POST` | `/api/settings/email/test` | Send a test email via Brevo |

---

## Further reading

- [deploy-pipeline.md](deploy-pipeline.md) — detailed deploy flow, both ssh_rsync and stella_deploy_api paths
- [monitoring.md](monitoring.md) — Stella health monitoring + WP Sites uptime
- [dotnet-azure.md](dotnet-azure.md) — deploying .NET apps to Azure App Service via Atlas
