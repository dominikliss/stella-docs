# Atlas architecture

Companion to [`README.md`](README.md). This page is the **code and hosting** map. For how Atlas talks to ddashboard, Stella, GitHub, and WP sites see [`../integration/system-overview.md`](../integration/system-overview.md).

---

## Hosting

| Item | Value |
|------|-------|
| Public URL | `https://atlas.foxcraft.digital` (old: `https://dev.atlas.foxcraft.digital`) |
| Host | Hetzner Cloud CX33 `atlas`, `2.31.42.13`, Ubuntu 26.04, user `atlas`. Runbook: [`server.md`](server.md) |
| App path | `/var/www/atlas` |
| PHP | 8.5.4-FPM (Ubuntu `8.5.4-0ubuntu1.5`, includes CVE-2026-109618/109619 backports; pool `atlas`, socket `/run/php/atlas.sock`) |
| Laravel | 12.61 |
| MySQL | 8.4 `atlas_prod` (migrated from managed-host `dev_atlas_foxcraft_digital`; docs previously said `atlas_dev`) |
| Redis | 8.0, cache + token verify cache (`CACHE_STORE=redis`) — localhost only, **not** the deploy queue |
| Cron | `/etc/cron.d/atlas` → `schedule:run` every minute as `atlas` |
| Queue | systemd `atlas-queue.service` — `queue:work database --queue=deployments` (not cron + `flock`) |

Not the same host as **ddashboard**. Atlas reuses the old managed-host SSH key Stella trusts as principal `ddashboard` (key comment `atlas-prod`). The managed host still holds that key.

---

## Backend layout

Business code lives under `app/Domain/{Name}/` (controllers, models, jobs, services). Laravel’s default `app/Http` is not the home for these modules.

| Domain | Responsibility |
|--------|----------------|
| `Auth` | Login proxy, token middleware, IP allowlist, `DdashboardTokenDriver` / `FakeTokenAuthDriver` |
| `Deploy` | Configs, servers, runs, rsync, Stella deploy-api client, SSH signer, GitHub branches, folder browser |
| `Monitoring` | Stella health-api + public-probe fallback; Atlas host-api health + optional self-deploy; Hetzner Cloud (atlas + edison backups); history + prune |
| `WpSites` | Site registry, uptime job, Atlas Connect login, down-alert mail, backup driver + explorer + browser decrypt — see [`atlas-connect/backups.md`](atlas-connect/backups.md). Infection scan driver, inventory, allowlist, verdict — [`atlas-connect/security-scan.md`](atlas-connect/security-scan.md). Staging from a full backup via Stella `wp-staging-api`. Fresh Atlas Connect zip (`GET /api/wp-sites/atlas-connect.zip`) |
| `Tools` | IMAP migration proxy to Stella `imap-sync`; TrackingTime import / open-task sync (ddashboard [`../stella-dashboard/atlas-client-lookup.md`](../stella-dashboard/atlas-client-lookup.md) for client stubs) |
| `Projects` | Imported customers/projects/tasks; project metadata edit; attachment download; one-way Trello push ([`projects.md`](projects.md)) |
| `Notifications` | Rules, Slack webhooks, in-app inbox, weekly hour-budget checker ([`notifications.md`](notifications.md)) |
| `Settings` | `app_settings` key/value (SSH path, Brevo, Hetzner Cloud token, backup S3 + X25519 vault, TrackingTime, Trello) |

`App\Providers\AtlasServiceProvider` binds `TokenAuthDriver`. With `ATLAS_DEMO_MODE=true` it binds `FakeTokenAuthDriver` instead so the UI can be exercised without ddashboard.

---

## Frontend layout

Entry: `resources/views/app.blade.php` → Vite → `resources/js/routes/index.tsx` (TanStack Router).

| Route | Page |
|-------|------|
| `/login` | `domains/auth/LoginPage` |
| `/backup-decrypt` | `domains/wp-sites/BackupDecryptPage` (no token; private key stays in the tab) |
| `/dashboard` | `domains/dashboard/DashboardPage` (+ `StellaServerStatus`, `AtlasServerStatus`, `EdisonServerStatus`) |
| `/deployments` | `domains/deployments/DeploymentsPage` (+ `ConfirmDeployModal` before each run) |
| `/wp-sites` | `domains/wp-sites/WpSitesPage` |
| `/monitoring/stella` | `domains/monitoring/StellaDetailPage` |
| `/monitoring/atlas` | `domains/monitoring/AtlasDetailPage` |
| `/monitoring/edison` | `domains/monitoring/EdisonDetailPage` |
| `/tools` | `domains/tools/ToolsPage` |
| `/tools/imap-migration` | `domains/tools/imap-migration/ImapMigrationPage` |
| `/tools/tracking-time-import` | `domains/tools/tracking-time-import/TrackingTimeImportPage` |
| `/tools/tracking-time-sync` | `domains/tools/tracking-time-sync/TrackingTimeSyncPage` |
| `/projects` | `domains/projects/ProjectsPage` (+ `ProjectEditForm` and `TrelloProjectSettings` on the detail view) |
| `/notifications` | `domains/notifications/NotificationsPage` |
| `/settings` | `domains/settings/SettingsPage` |

API calls go through `resources/js/lib/api-client.ts` with the bearer token from `lib/token-store.ts` (access + refresh). The browser never calls Stella, GitHub, or the Atlas Connect plugin directly.

Shell / nav: `resources/js/components/AppShell.tsx`, `SidebarDrawer.tsx`.

---

## Request pipeline

```
HTTP
  → AllowedIpMiddleware     (global prepend; GET /api/health returns early inside the class)
  → TokenAuthMiddleware     (auth.token group)
       1. Laravel actingAs (tests)
       2. Cache hit  token:{sha256(bearer)} → local User
       3. Cache miss → DdashboardTokenDriver::verify()
            GET {ATLAS_DDASHBOARD_URL}/dls/v1/auth/verify
       4. firstOrCreate User, sync Spatie roles, bind team
  → Domain controller
```

Login / refresh / logout skip token middleware (still IP-filtered) and proxy to ddashboard. See [`auth.md`](auth.md).

---

## Persistence

| Table / store | Purpose |
|---------------|---------|
| `users`, `teams`, Spatie permission tables | Local user projection of ddashboard identity |
| `personal_access_tokens` | Sanctum leftover — **not** the live Atlas API auth path |
| `app_settings` | SSH key path/passphrase, Brevo keys, Hetzner Cloud token, TrackingTime, Trello key/token (encrypted), alert recipient |
| `deploy_configs`, `deploy_destinations`, `deploy_servers` | What/where to deploy (`deploy_destinations.is_active` defaults true) |
| `deploy_runs`, `deploy_run_destinations` | Run history + per-target logs / `stella_job_id` |
| `stella_health_checks`, `stella_service_uptime_logs` | 5-min Stella snapshots (30-day retention) |
| `wp_sites`, `wp_site_uptime_logs` | Site registry + 7-day uptime |
| `wp_site_backups` | Atlas Connect backup jobs, progress, manifest hash |
| `wp_site_scans`, `wp_site_scan_files`, `wp_site_scan_allowlist_entries` | Infection-scan jobs, inventory, allowlist |
| `wp_site_stagings` | Stella staging builds from a full backup |
| `imap_migration_jobs` | Atlas-side IMAP history (hosts/users only — **no passwords**) |
| `customers`, `projects`, `task_lists`, `tasks`, `task_checklist_items`, `task_comments`, `time_entries`, `attachments` | TrackingTime import (one-way). Trello ids live on projects/tasks/check items/attachments. `projects.max_hours_per_week` / `trello_pushed_hash` and `tasks.skip_trello` are Atlas-local |
| `slack_destinations`, `notification_rules`, `notification_rule_channels`, `notification_rule_projects`, `notification_bindings`, `notification_events`, `notification_deliveries` | Notification rules, Slack webhooks (encrypted), fires + per-channel deliveries |
| `tracking_time_import_jobs` | Import / open-task sync history |
| Redis | Cache, including hashed access tokens after login |
| Database queue `jobs` / `deployments` | `DeployRunJob`, `ImportTrackingTimeJob`, `PushProjectToTrelloJob` |

---

## Jobs vs scheduler

| Mechanism | Transport | Used for |
|-----------|-----------|----------|
| systemd `atlas-queue.service` | `queue:work database --queue=deployments --sleep=3 --timeout=7200 --max-time=3500` | `DeployRunJob`, `ImportTrackingTimeJob`, `PushProjectToTrelloJob` |
| Laravel scheduler (`dispatchSync`) | In-process during `schedule:run` | `CheckStellaHealth`, `CheckAtlasHealth`, `CheckWpSiteUptime`, `wp-sites:drive-backups`, `wp-sites:drive-scans`. Daily `wp-sites:run-daily-scans` at 04:00 and `wp-sites:run-daily-backups` at 05:00 Vienna |
| Laravel scheduler (command) | Hourly `projects:sync-tracking-time-open` | Open-task TrackingTime sync; queues Trello push when the project is mapped **and** Atlas content changed since the last successful push |
| Laravel scheduler (command) | Hourly `notifications:check-week-hours` | Fire weekly hour-budget notifications for watched projects that crossed a threshold |
| Laravel scheduler (command) | Daily 07:00 `Europe/Vienna` `notifications:check-missing-comments` | Refresh tasks with uncommented time since 2026-10-19 (including done), then notify |

Do not assume Horizon / Redis queues are driving deploys. Production **hard-codes** `database` and the `deployments` queue. `scripts/queue-deployments.sh` no longer applies on `atlas`.

---

## Signing Stella requests

`App\Domain\Deploy\Services\StellaSshSigner` is shared by deploy-api and health-api:

```
payload  = "{action}:{unix_timestamp}"
signature = ssh-keygen -Y sign -f {key} -n deploy  {payload}
POST JSON { payload, signature }
```

| Caller | Action prefix |
|--------|----------------|
| `StellaDeployApiService::listApps` | `list-apps:` |
| `StellaDeployApiService::trigger` | `deploy-{app}:` |
| `StellaHealthApiClient::fetchSystemHealth` | `system-health:` |

Stella verifies with principal `ddashboard` and namespace `deploy` (health-api hardcodes both in `main.py`; Atlas sends no principal). Replay window is 60 seconds on Stella.

deploy-api and health-api each have their **own** `allowed_signers` file. health-api’s file has two `ddashboard` lines (`atlas-prod` and `atlas-dev`). deploy-api’s file has only the production key. See [`../stella-server/health-api.md`](../stella-server/health-api.md) and [`dev-instance.md`](dev-instance.md).

---

## Demo mode

`ATLAS_DEMO_MODE=true` swaps the token driver for `FakeTokenAuthDriver`. Comments in `AtlasServiceProvider` reserve the same switch for future fake SSH / Atlas Connect clients. Use it to click through the UI without live infrastructure — not as a production config.

---

## Tests

Pest feature tests under `tests/Feature/{Auth,Deploy,Projects,Settings}/` and unit tests under `tests/Unit/`. Auth tests point `atlas.ddashboard_url` at a fake host and mock HTTP. Trello tests fake `api.trello.com`.

---

## What Atlas is not

- Not a WordPress theme and not a ddashboard module
- Not the Stella UI
- Not an identity provider
- Not responsible for Edison / Cursor agents (Cloud API reads server + backup status only; no agent control)
- Not the mailbox **importer** (ddashboard `dls_mail_*`); Atlas only starts **mailbox copies** on Stella
