# Atlas Deploy Pipeline

## Deploy types

A `DeployConfig` has a `type` field that controls which deploy path `DeployRunJob` takes:

| Type | Path | When to use |
|---|---|---|
| `ssh_rsync` | Clone from GitHub → rsync to SSH servers | PHP/WordPress apps, static files — source is deployed as-is |
| `stella_deploy_api` | Signed HTTP trigger to Stella deploy-api → poll for completion | .NET apps, Docker-based apps where Stella handles the actual build/restart |

---

## `ssh_rsync` path

### Triggering a run

A run is created via `POST /api/deploy/runs` with a `deploy_config_id`. This:

1. Creates a `DeployRun` record (status: `pending`)
2. Creates a `DeployRunDestination` row for each destination in the config (status: `pending`)
3. Dispatches `DeployRunJob` onto the `deployments` database queue

The job executes when the queue worker picks it up (see [README.md — Queue worker](README.md#queue-worker)).

---

## Job execution — step by step

### 1. Clone from GitHub

```bash
git clone --depth=1 -b {branch} git@github.com:{owner}/{repo}.git /tmp/atlas-deploy-run-{id}/
```

Uses `GIT_SSH_COMMAND` with the configured SSH key. Shallow clone (`--depth=1`) — no history, just the tip of the branch.

If clone fails, **all destinations** are immediately marked `error` and the run ends. No partial deploys.

After clone, Atlas reads `git rev-parse HEAD` and `git log -1 --pretty=%s` to capture the commit SHA and message, stored on the run record.

### 2. rsync to each destination (in order)

For each `DeployRunDestination` (ordered by `sort_order`):

```bash
rsync -az --delete --exclude=.git --exclude=.cursor --exclude=docs \
  -e "ssh -i {key} -o BatchMode=yes -o StrictHostKeyChecking=accept-new -o ConnectTimeout=30 -p {port}" \
  /tmp/atlas-deploy-run-{id}/ \
  {user}@{host}:{destination_path}/
```

Before rsync, Atlas SSHes in to `mkdir -p {destination_path}` if it doesn't exist.  
After rsync, Atlas SSHes in to set permissions: `find ... -type d -exec chmod 755` / `find ... -type f -exec chmod 644`.

### 3. Post-deploy endpoint (optional)

If the destination has a `post_deploy_endpoint` URL, Atlas calls it via HTTP POST after a successful rsync. The response is appended to the destination's log. If the endpoint returns non-2xx, the destination is marked `error` even though the rsync succeeded.

### 4. Cleanup

`rm -rf /tmp/atlas-deploy-run-{id}/` — always runs, even on error or cancellation.

---

## Destination delay (cooldown between destinations)

Controlled by `config/deploy.php`:

| Key | Default | Purpose |
|---|---|---|
| `destination_delay_every` | `5` | Apply a delay before every Nth destination |
| `destination_delay_seconds` | `30` | How long to wait |

The job **releases itself back to the queue** for the delay duration rather than sleeping — keeps the queue worker free. The delayed destination is marked `waiting` in the UI while it waits.

Example: `delay_every=5, delay_seconds=30` → destinations 1–5 deploy immediately, destination 6 waits 30s, 7–10 immediately, 11 waits 30s, etc.

---

## Cancellation

`POST /api/deploy/runs/{id}/cancel` sets the run to `cancelled`. The job checks for this between destinations — it will finish the currently-in-progress destination, then stop. Destinations not yet started are left as `pending` (not `cancelled`).

---

## Final status

| Condition | Status |
|---|---|
| All destinations `success` | `success` |
| All destinations `error` | `error` |
| Mixed | `partial` |

---

## Timeouts

The job timeout scales with the number of destinations:

```
max(600, 600 + (max_delays × delay_seconds) + (destination_count × 300))
```

Minimum 10 minutes. Each destination adds 5 minutes of buffer, plus time for any configured delays.

The queue worker itself runs with `--timeout=7200` (2 hours hard cap) and `--max-time=3500` (~58 minutes max runtime per worker process).

---

## Error handling

| Failure point | Behaviour |
|---|---|
| Clone fails | All destinations → `error`, run → `error`, tmp cleaned up |
| SSH connect fails | That destination → `error`, others continue |
| rsync fails | That destination → `error`, others continue |
| Post-deploy endpoint fails | That destination → `error` (rsync output preserved in log) |
| Job exception (uncaught) | `DeployRunJob::failed()` — run + all pending/running destinations → `error` |

---

## Logs

Each `DeployRunDestination` stores a plain-text `log` column with step-by-step output prefixed `[atlas]`. Example:

```
[atlas] Connected to deploy@203.0.113.10
[atlas] Commit: a3f2c91
[atlas] Preparing destination: /var/www/html
[atlas] Syncing files via rsync...
sending incremental file list
...
[atlas] Setting permissions...
[atlas] Deployment successful.
```

Logs are visible per-destination in the Atlas UI run detail view.

---

## `stella_deploy_api` path

Used when `DeployConfig.type === 'stella_deploy_api'`. No GitHub clone, no rsync, no SSH destinations. Atlas delegates all build/deploy logic to the Stella deploy-api running on the Stella server.

### Config fields

| Field | Description |
|---|---|
| `stella_deploy_url` | Base URL of the Stella deploy-api (e.g. `https://stella-deployment-api.foxcraft.digital`) |
| `stella_deploy_app` | App name registered in the Stella deploy-api (e.g. `advoapp-dev`) |

`config('deploy.stella_api_url')` (from env `STELLA_DEPLOY_API_URL`) overrides `stella_deploy_url` when set — useful for pointing all Stella-type configs at the same host without per-config URL maintenance.

### Job execution

```
DeployRunJob::handleStellaApiDeploy()
  1. Sign payload "deploy-{app}:{timestamp}" with Atlas SSH key
  2. POST {baseUrl}/deploy/{app}  → { payload, signature }
     ← { job_id }
  3. Store stella_job_id on DeployRunDestination (allows resume on worker restart)
  4. Poll GET {baseUrl}/deploy/{app}/status/{job_id} every 5 s
     ← { status, current_step, log: [...], commit_sha, commit_message }
     - Append new log lines to DeployRunDestination.log in real time
     - Done when status ∈ { success, failed, error }
     - Cancelled when DeployRun.status === 'cancelled' (checked each poll cycle)
     - Times out after 30 min (360 × 5 s) → both destination and run set to error
```

There is always exactly **one** `DeployRunDestination` for a Stella-type run (a synthetic row that holds the live log and `stella_job_id`).

### Signature scheme

Same SSH-signature mechanism as health-api — see [`../stella-server/deploy-api.md`](../stella-server/deploy-api.md). Payload format: `"deploy-{appName}:{unixTimestamp}"`, namespace `deploy`.

### Error handling (stella_deploy_api)

| Failure point | Behaviour |
|---|---|
| Signing fails | Destination + run → `error` immediately |
| HTTP request fails (curl error) | Destination + run → `error` immediately |
| Stella returns non-200/201 | Destination + run → `error` immediately |
| Poll returns `failed`/`error` | Destination + run → `error`, log preserved |
| 30-min poll timeout | Destination + run → `error` |
| Job exception | `DeployRunJob::failed()` — same behaviour as ssh_rsync |
| Worker restart mid-poll | `stella_job_id` already stored on the destination row → job resumes polling on restart (no re-trigger) |
