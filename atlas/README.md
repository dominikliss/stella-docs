# Atlas

**Atlas** is the internal deployment management platform for foxcraft.digital projects. It provides a web UI and API to orchestrate deployments from GitHub to production servers over SSH.

**Live at:** `https://dev.atlas.foxcraft.digital`  
**Stack:** Laravel 12, React + Inertia.js, MySQL, database-backed queue  
**Repo:** `git@github.com:foxcraft/atlas.git`

---

## What it does

Atlas clones a GitHub repository, rsyncs the files to one or more SSH servers, and optionally calls post-deploy HTTP endpoints. It has no build step of its own — it deploys source files as-is, which makes it a natural fit for PHP/WordPress apps. For compiled apps (.NET, etc.) see [dotnet-azure.md](dotnet-azure.md).

---

## Data model

```
DeployConfig          — what to deploy (GitHub repo + branch + environment)
  └── DeployDestination — where (server + path + optional endpoint URLs)
        └── DeployServer  — the SSH target (host, port, user)

DeployRun             — one triggered deploy (status, commit SHA, log per destination)
  └── DeployRunDestination — per-server result of that run
```

A single `DeployConfig` can have multiple `DeployDestination`s — e.g. deploy the same repo to staging and production in one run, with a configurable cooldown delay between destinations.

---

## Architecture

```
Browser / API caller
      │
      ▼
Laravel (Sanctum token auth)
  POST /api/deploy/runs         → creates DeployRun + queues DeployRunJob
  GET  /api/deploy/runs/{id}    → poll status + per-destination logs
      │
      ▼
Database queue (deployments queue)
      │
      ▼
DeployRunJob (queue worker)
  1. git clone --depth=1  (GitHub SSH → /tmp/atlas-deploy-run-{id}/)
  2. rsync -az --delete    (local clone → each destination server over SSH)
  3. POST post_deploy_endpoint (optional, per destination)
  4. cleanup tmp dir
```

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

All API routes require a **Sanctum bearer token**:
```
Authorization: Bearer {token}
```

Tokens are managed in the Atlas UI (Settings → API tokens) or via `php artisan sanctum:prune-expired`.

---

## SSH key

Atlas uses a **single SSH key** for two things:
1. Cloning private GitHub repositories
2. SSH-ing to destination servers for rsync

Configured in: **Settings → GitHub** (stored in `app_settings` table under key `deploy_ssh_key_path`). Falls back to `config/deploy.php` → `~/.ssh/id_rsa`.

The key's **public key** must be added to:
- GitHub (deploy key or personal account key) — for cloning
- Each destination server's `~/.ssh/authorized_keys` — for rsync

---

## Key API endpoints

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/api/deploy/configs` | List all deploy configurations |
| `POST` | `/api/deploy/configs` | Create a deploy config |
| `GET` | `/api/deploy/servers` | List SSH servers |
| `POST` | `/api/deploy/servers` | Create an SSH server |
| `POST` | `/api/deploy/runs` | Trigger a deploy run |
| `GET` | `/api/deploy/runs/{id}` | Poll run status + logs |
| `GET` | `/api/deploy/runs/active` | Any currently-running runs |
| `POST` | `/api/deploy/runs/{id}/cancel` | Cancel a running deploy |
| `POST` | `/api/deploy/configs/{id}/test` | Test SSH connectivity for all destinations |

---

## Further reading

- [deploy-pipeline.md](deploy-pipeline.md) — detailed deploy flow, timeouts, delays, error handling
- [dotnet-azure.md](dotnet-azure.md) — deploying .NET apps to Azure App Service via Atlas
