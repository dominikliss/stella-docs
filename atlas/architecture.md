# Atlas architecture

Companion to [`README.md`](README.md). This page is the **code and hosting** map. For how Atlas talks to ddashboard, Stella, GitHub, and WP sites see [`../integration/system-overview.md`](../integration/system-overview.md).

---

## Hosting

| Item | Value |
|------|-------|
| Public URL | `https://dev.atlas.foxcraft.digital` |
| Host | Hetzner managed, Linux user `foxcrr` |
| App path | `/usr/home/foxcrr/public_html/dev.atlas.foxcraft.digital` |
| PHP | 8.2+ (Laravel 12) |
| MySQL | `atlas_dev` |
| Redis | cache + token verify cache (`CACHE_STORE=redis`) |
| Cron | `schedule:run` every minute; `scripts/queue-deployments.sh` for the deploy queue |
| Worker log | `/usr/home/foxcrr/logs/queue-worker.log` |
| Queue lock | `/usr/home/foxcrr/tmp/atlas-deploy-queue.lock` |

Same managed account as **ddashboard**. They share the host SSH key that Stella trusts as principal `ddashboard`.

---

## Backend layout

Business code lives under `app/Domain/{Name}/` (controllers, models, jobs, services). Laravel’s default `app/Http` is not the home for these modules.

| Domain | Responsibility |
|--------|----------------|
| `Auth` | Login proxy, token middleware, IP allowlist, `DdashboardTokenDriver` / `FakeTokenAuthDriver` |
| `Deploy` | Configs, servers, runs, rsync, Stella deploy-api client, SSH signer, GitHub branches, folder browser |
| `Monitoring` | Signed health-api client, public-probe fallback, history + prune |
| `WpSites` | Site registry, uptime job, Atlas Connect login, down-alert mail |
| `Tools` | IMAP migration proxy to Stella `imap-sync` |
| `Settings` | `app_settings` key/value (SSH path, Brevo) |

`App\Providers\AtlasServiceProvider` binds `TokenAuthDriver`. With `ATLAS_DEMO_MODE=true` it binds `FakeTokenAuthDriver` instead so the UI can be exercised without ddashboard.

---

## Frontend layout

Entry: `resources/views/app.blade.php` → Vite → `resources/js/routes/index.tsx` (TanStack Router).

| Route | Page |
|-------|------|
| `/login` | `domains/auth/LoginPage` |
| `/dashboard` | `domains/dashboard/DashboardPage` (+ `StellaServerStatus`) |
| `/deployments` | `domains/deployments/DeploymentsPage` |
| `/wp-sites` | `domains/wp-sites/WpSitesPage` |
| `/monitoring/stella` | `domains/monitoring/StellaDetailPage` |
| `/tools` | `domains/tools/ToolsPage` |
| `/tools/imap-migration` | `domains/tools/imap-migration/ImapMigrationPage` |
| `/settings` | `domains/settings/SettingsPage` |

API calls go through `resources/js/lib/api-client.ts` with the bearer token from `lib/token-store.ts` (access + refresh). The browser never calls Stella, GitHub, or `dl-connect` directly.

Shell / nav: `resources/js/components/AppShell.tsx`, `SidebarDrawer.tsx`.

---

## Request pipeline

```
HTTP
  → AllowedIpMiddleware     (all API routes except GET /api/health)
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
| `app_settings` | SSH key path/passphrase, Brevo keys and alert recipient |
| `deploy_configs`, `deploy_destinations`, `deploy_servers` | What/where to deploy |
| `deploy_runs`, `deploy_run_destinations` | Run history + per-target logs / `stella_job_id` |
| `stella_health_checks`, `stella_service_uptime_logs` | 5-min Stella snapshots (30-day retention) |
| `wp_sites`, `wp_site_uptime_logs` | Site registry + 7-day uptime |
| `imap_migration_jobs` | Atlas-side IMAP history (hosts/users only — **no passwords**) |
| Redis | Cache, including hashed access tokens after login |
| Database queue `jobs` / `deployments` | `DeployRunJob` |

---

## Jobs vs scheduler

| Mechanism | Transport | Used for |
|-----------|-----------|----------|
| `scripts/queue-deployments.sh` | `queue:work database --queue=deployments --stop-when-empty` | `DeployRunJob` only |
| Laravel scheduler (`dispatchSync`) | In-process during `schedule:run` | `CheckStellaHealth`, `CheckWpSiteUptime` |

Do not assume Horizon / Redis queues are driving deploys. The cron script **hard-codes** `database` and the `deployments` queue.

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

Stella verifies against `allowed_signers` with principal `ddashboard` and namespace `deploy`. Replay window is 60 seconds on Stella.

---

## Demo mode

`ATLAS_DEMO_MODE=true` swaps the token driver for `FakeTokenAuthDriver`. Comments in `AtlasServiceProvider` reserve the same switch for future fake SSH / Atlas Connect clients. Use it to click through the UI without live infrastructure — not as a production config.

---

## Tests

Pest feature tests under `tests/Feature/{Auth,Deploy}/` and unit tests under `tests/Unit/Deploy/`. Auth tests point `atlas.ddashboard_url` at a fake host and mock HTTP.

---

## What Atlas is not

- Not a WordPress theme and not a ddashboard module
- Not the Stella UI
- Not an identity provider
- Not responsible for Edison / Cursor agents (no code path today)
- Not the mailbox **importer** (ddashboard `dls_mail_*`); Atlas only starts **mailbox copies** on Stella
