# Atlas architecture

Companion to [`README.md`](README.md). This page is the **code and hosting** map. For how Atlas talks to ddashboard, Stella, GitHub, and WP sites see [`../integration/system-overview.md`](../integration/system-overview.md). SPA chrome follows [`../design-language/SPEC.md`](../design-language/SPEC.md).

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
| `Auth` | Local password + TOTP login, Atlas-issued tokens, user admin, IP allowlist; leftover `DdashboardTokenDriver` / `FakeTokenAuthDriver` verify fallback; OIDC authorization-code + PKCE for Edison (`Domain/Auth/Oidc`) |
| `Deploy` | Configs, servers, runs, rsync, Stella deploy-api client, SSH signer, GitHub branches, folder browser, source zip (`DeploySourcePackager`) |
| `Monitoring` | Stella health-api + public-probe fallback; Atlas host-api health + optional self-deploy; Hetzner Cloud (atlas + edison backups); history + prune |
| `WpSites` | Site registry, uptime job, Atlas Connect login, down-alert mail, backup driver + explorer + browser decrypt — see [`atlas-connect/backups.md`](atlas-connect/backups.md). Infection scan driver, inventory, allowlist, verdict — [`atlas-connect/security-scan.md`](atlas-connect/security-scan.md). Staging from a full backup is planned as `create` + `restore` on Stella [`stack-api`](../stella-server/stack-api.md); `wp-staging-api` will not be deployed. Fresh Atlas Connect zip (`GET /api/wp-sites/atlas-connect.zip`) |
| `Tools` | IMAP migration proxy to Stella `imap-sync`; TrackingTime import / open-task sync (ddashboard [`../stella-dashboard/atlas-client-lookup.md`](../stella-dashboard/atlas-client-lookup.md) for client stubs) |
| `Projects` | Imported customers/projects/tasks; project metadata edit; attachment download; one-way Trello push ([`projects.md`](projects.md)) |
| `Notifications` | Rules, Slack webhooks, in-app inbox, weekly hour-budget checker ([`notifications.md`](notifications.md)) |
| `Stacks` | Stella stack-api client: list / get / restart / logs / jobs / plan / apply / update / reconcile / people / people-sync / fix-permissions / delete / trash / restore. Admin only. Hidden unless `STELLA_STACK_API_ENABLED`. Local `stack_actions` history. **On** atlas.dev (create live 2026-11-08); **off** on prod. UI: [`stacks.md`](stacks.md) |
| `Settings` | `app_settings` key/value (SSH path, stack-api SSH path, Brevo, Hetzner Cloud token, backup S3 + X25519 vault, TrackingTime, Trello, ddashboard service token) |

`App\Providers\AtlasServiceProvider` binds `TokenAuthDriver` for the leftover ddashboard-bearer fallback. With `ATLAS_DEMO_MODE=true` it binds `FakeTokenAuthDriver`. Live login uses local passwords and `AuthTokenService`.

---

## Frontend layout

Entry: `resources/views/app.blade.php` → Vite → `resources/js/routes/index.tsx` (TanStack Router).

| Route | Page |
|-------|------|
| `/login` | `domains/auth/LoginPage` (honors `?continue=/oauth/authorize?…` after 2FA) |
| `/oauth/authorize` | `domains/auth/OidcAuthorizePage` (Edison SSO; issues a code, not an Atlas bearer) |
| `/forgot-password` | `domains/auth/ForgotPasswordPage` |
| `/reset-password` | `domains/auth/ResetPasswordPage` |
| `/profile` | `domains/auth/ProfilePage` |
| `/users` | `domains/auth/UsersPage` (admin) |
| `/backup-decrypt` | `domains/wp-sites/BackupDecryptPage` (no token; private key stays in the tab) |
| `/dashboard` | `domains/dashboard/DashboardPage` (+ `StellaServerStatus`, `AtlasServerStatus`, `EdisonServerStatus`) |
| `/deployments` | `domains/deployments/DeploymentsPage` (+ `ConfirmDeployModal` before each run) |
| `/wp-sites` | `domains/wp-sites/WpSitesPage` |
| `/stacks` | `domains/stacks/StacksPage` (nav hidden unless `STELLA_STACK_API_ENABLED`; card grid grouped by type with live preview + restart + drift badge; search; New stack: Plan → Create; Stacks / People / Trash tabs) |
| `/stacks/people` | `domains/stacks/StackPeoplePage` (people list, out-of-sync stacks, Sync / Sync all) |
| `/stacks/trash` | `domains/stacks/StackTrashPage` (trash cards, purge after, restore) |
| `/stacks/$name` | same `StacksPage` with `StackDetailPanel` sidebar (containers, logs 50/200/1000, restart, edit / reconcile when managed, delete when managed, recent jobs, history) |
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
       2. Atlas access token (auth_access_tokens / AuthTokenService)
       3. Leftover: DdashboardTokenDriver::verify()
            GET {ATLAS_DDASHBOARD_URL}/dls/v1/auth/verify
       4. bind local User + team
  → Domain controller
```

Login / refresh / logout / forgot / reset skip token middleware (still IP-filtered) and use Atlas-issued tokens. OIDC `/oauth/token`, `/oauth/jwks`, and discovery are also unauthenticated (client secret / PKCE). `POST /api/oauth/authorize` uses the Atlas SPA bearer. See [`auth.md`](auth.md).

---

## Persistence

| Table / store | Purpose |
|---------------|---------|
| `users`, `teams`, Spatie permission tables | Atlas identity (password, TOTP, `is_active`, roles). `ddashboard_user_id` is leftover from the old proxy |
| `auth_access_tokens`, `auth_refresh_tokens` | Atlas-issued API sessions (hashed). Sanctum `personal_access_tokens` is leftover |
| `app_settings` | SSH key path/passphrase, `stack_api_ssh_key_path`, Brevo keys, Hetzner Cloud token, TrackingTime, Trello key/token, ddashboard service token (encrypted), alert recipient |
| `deploy_configs`, `deploy_destinations`, `deploy_servers` | What/where to deploy (`deploy_destinations.is_active` defaults true) |
| `deploy_runs`, `deploy_run_destinations` | Run history + per-target logs / `stella_job_id` |
| `stella_health_checks`, `stella_service_uptime_logs` | 5-min Stella snapshots (30-day retention) |
| `wp_sites`, `wp_site_uptime_logs` | Site registry + 7-day uptime |
| `wp_site_backups` | Atlas Connect backup jobs, progress, manifest hash |
| `wp_site_scans`, `wp_site_scan_files`, `wp_site_scan_allowlist_entries` | Infection-scan jobs, inventory, allowlist |
| `wp_site_stagings` | Stella staging builds from a full backup |
| `stack_actions` | Atlas-side history for restart, apply, update, reconcile, people-sync, fix-permissions, delete, and trash-restore (`user_id`, stack, service, result, detail, created_at). Queued rows store the job id in `detail` and settle to `ok` / `error`. stack-api has its own audit log |
| `imap_migration_jobs` | Atlas-side IMAP history (hosts/users only — **no passwords**) |
| `customers`, `projects`, `task_lists`, `tasks`, `task_checklist_items`, `task_comments`, `time_entries`, `attachments` | TrackingTime import (one-way). Trello ids live on projects/tasks/check items/attachments. `projects.max_hours_per_week` / `trello_hours_custom_field_id` / `trello_deadline_custom_field_id` / `trello_pushed_hash` and `tasks.skip_trello` are Atlas-local. `tasks.deadline` is the TrackingTime **Deadline** custom field (`set_custom_fields`) |
| `slack_destinations`, `notification_rules`, `notification_rule_channels`, `notification_rule_projects`, `notification_bindings`, `notification_events`, `notification_deliveries` | Notification rules, Slack webhooks (encrypted), fires + per-channel deliveries |
| `tracking_time_import_jobs` | Import / open-task sync history |
| Redis | Cache, including hashed access tokens after login |
| Database queue `jobs` / `deployments` | `DeployRunJob`, `ImportTrackingTimeJob`, `PushProjectToTrelloJob` |

---

## Jobs vs scheduler

| Mechanism | Transport | Used for |
|-----------|-----------|----------|
| systemd `atlas-queue.service` | `queue:work database --queue=deployments --sleep=3 --timeout=7200 --max-time=3500` | `DeployRunJob`, `ImportTrackingTimeJob`, `PushProjectToTrelloJob` |
| Laravel scheduler (`dispatchSync`) | In-process during `schedule:run` | `CheckStellaHealth`, `CheckAtlasHealth`, `CheckWpSiteUptime`, `wp-sites:drive-backups`, `wp-sites:drive-scans`. `wp-sites:run-daily-scans` at 04:00 and 16:00, daily `wp-sites:run-daily-backups` at 05:00 Vienna |
| Laravel scheduler (command) | Hourly `projects:sync-tracking-time-open` | Open-task TrackingTime sync; queues Trello push when the project is mapped **and** Atlas content changed since the last successful push |
| Laravel scheduler (command) | Hourly `notifications:check-week-hours` | Fire weekly hour-budget notifications for watched projects that crossed a threshold |
| Laravel scheduler (command) | Daily 07:00 `Europe/Vienna` `notifications:check-missing-comments` | Refresh tasks with uncommented time since 2026-10-19 (including done), then notify |

Do not assume Horizon / Redis queues are driving deploys. Production **hard-codes** `database` and the `deployments` queue. `scripts/queue-deployments.sh` no longer applies on `atlas`.

---

## Signing Stella requests

`App\Domain\Deploy\Services\StellaSshSigner` is shared. `sign($payload, $namespace = 'deploy', $keyPath = null)` — existing deploy-api / health-api callers keep namespace `deploy` and the deploy key.

```
payload  = "{action}:{unix_timestamp}"                 # deploy-api / health-api
payload  = "{action}:{unix_ts}:{sha256(request_json)}" # stack-api / wp-staging
signature = ssh-keygen -Y sign -f {key} -n {namespace}  {payload}
```

| Caller | Action prefix | Namespace | Key |
|--------|----------------|-----------|-----|
| `StellaDeployApiService::listApps` | `list-apps:` | `deploy` | `deploy_ssh_key_path` |
| `StellaDeployApiService::trigger` | `deploy-{app}:` | `deploy` | `deploy_ssh_key_path` |
| `StellaHealthApiClient::fetchSystemHealth` | `system-health:` | `deploy` | `deploy_ssh_key_path` |
| `StackApiClient` | `stack-ping` / `stack-list` / `stack-get` / `stack-restart` / `stack-logs` / `job-get` / `job-list` / `stack-plan` / `stack-apply` / `people-list` / `stack-fix-permissions` / `stack-delete` / `stack-trash-list` / `stack-trash-restore` | `stack` | `stack_api_ssh_key_path` (fallback `STELLA_STACK_API_KEY_PATH`) |

stack-api POST body is `{payload, signature, request}` with header `X-Stack-Token`. Never log the token, signature, or key contents. Stella verifies stack-api with namespace `stack`. Prod principal `atlas-prod` (`/home/atlas/.ssh/atlas-stack-api`, flag still off). atlas.dev principal `atlas-dev` (`/var/www/.ssh/id_ed25519`) — visible = `STACK_API_SANDBOX_STACKS` ∪ stacks it owns (allowlist currently `dominikliss.foxcraft.digital`, `sandbox.staging.foxcraft.digital`; it may create any valid `*.staging.foxcraft.digital` name). Replay window is 60 seconds on Stella. Every request JSON includes a random `nonce` (Ed25519 signatures are deterministic).

deploy-api and health-api each have their **own** `allowed_signers` file. health-api’s file has two `ddashboard` lines (`atlas-prod` and `atlas-dev`). deploy-api’s file has only the production key. See [`../stella-server/health-api.md`](../stella-server/health-api.md) and [`dev-instance.md`](dev-instance.md).

---

## Demo mode

`ATLAS_DEMO_MODE=true` swaps the token driver for `FakeTokenAuthDriver`. Comments in `AtlasServiceProvider` reserve the same switch for future fake SSH / Atlas Connect clients. Use it to click through the UI without live infrastructure — not as a production config.

---

## Tests

Pest feature tests under `tests/Feature/{Auth,Deploy,Projects,Settings,Stacks}/` and unit tests under `tests/Unit/`. Auth tests create local users (password + TOTP). Trello tests fake `api.trello.com`. Stack tests (`Http::fake` stack-api) pass on atlas.dev (11 tests, 2026-11-04).

---

## What Atlas is not

- Not a WordPress theme and not a ddashboard module
- Not the Stella UI
- Not the identity provider for **ddashboard**. It **is** the IdP for Edison (OIDC code + PKCE; no Atlas API bearer sharing)
- Not responsible for Edison / Cursor agents (Cloud API reads server + backup status only; no agent control)
- Not the mailbox **importer** (ddashboard `dls_mail_*`); Atlas only starts **mailbox copies** on Stella
