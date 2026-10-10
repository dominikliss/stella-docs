# Atlas Monitoring

Atlas monitors three things on a 5-minute cron loop:

1. **Stella server health** — signed requests to Stella's health-api; dynamic container list, Caddyfile certs, disk/CPU/RAM/Ollama, backup
2. **Atlas host health** — signed requests to the Atlas host API (`https://api.atlas.foxcraft.digital`); systemd units, certs, disk/CPU/RAM, optional backup
3. **WP site uptime** — plain HTTPS HEAD ping to each registered WordPress site

---

## Stella health monitoring

### How it works

`CheckStellaHealth` runs every 5 minutes via the Laravel scheduler. It delegates to `StellaHealthRecorder`, which tries two paths in order:

```
StellaHealthRecorder::run()
  1. If STELLA_HEALTH_API_URL is set:
       Sign "system-health:{timestamp}" with Atlas SSH key
       POST {url}/system/health  → { payload, signature }
       ← full health JSON (containers, certs, disk, ollama, backup, …)
       Normalize into per-service rows
  2. If health-api unreachable / not configured:
       StellaHealthService::check() — concurrent HTTP probes to each public endpoint
       (deploy-api, imap-sync, stella-api, ollama, caddy, advoapp, osgar-datahub)
```

Source is recorded as `health-api`, `probe`, or `probe-fallback` on the `StellaHealthCheck` row.

### Data model

```
StellaHealthCheck
  id, status (ok|degraded|down), source, response_ms, payload (JSON), error, checked_at

StellaServiceUptimeLog
  id, stella_health_check_id (FK cascade), service_id, service_label, category,
  status, response_ms, message, meta (JSON), checked_at
```

**Retention:** records older than 30 days are pruned after each check (`STELLA_HEALTH_RETENTION_DAYS`, default 30).

### Service categories (from health-api path)

The Stella health service list is dynamic as of 2026-10-10. health-api no longer ships a hardcoded container or cert list — Atlas renders whatever comes back.

| Category | Services |
|---|---|
| `system` | `system:docker_daemon`, `system:ollama_models`, `system:ollama_resources`, `system:cpu_ram`, `system:disk` |
| `container` | Core Stella containers from `CORE_CONTAINERS` in `health.py` (`caddy`, `stella-api`, `imap-sync`, `deploy-api`, `ollama`, `backup`) |
| `app` | Every other discovered container. Atlas then groups them by stack (see below). |
| `backup` | `system:backup` — current restic status (not a container) |
| `http` | `http:stella-api`, `http:ollama`, plus any `health.http` probes |
| `cert` | One row per site block parsed from the live Caddyfile |

**App grouping (Atlas UI, 2026-10-10):** hosted containers stay category `app`. `normalizeServices()` writes `meta.app_type` and `meta.app_project`.

| `app_type` | Name prefixes |
|---|---|
| `dotnet` | `advoapp*`, `osgar-datahub*` |
| `laravel` | `atlas.dev*` |
| `wordpress` | `*dominikliss*`, `wordpress*` |
| `other` | anything else (shown only when present) |

**Project / subdomain:** strip a trailing `-dev` / `-db` / `-ssh` / `-queue` / `-scheduler`. So `advoapp-dev` + `advoapp-ssh` are one project (`advoapp`); `osgar-datahub-*` is one; `atlas.dev.foxcraft.digital-*` is one; `dominikliss.foxcraft.digital-*` is one.

Tiles show **project counts**, not containers (Dotnet `2/2`, Laravel `1/1`, WordPress `1/1`). A project is up only if every container in it is ok. The uptime tables group rows by that project name under the stack heading.

A new app under `/opt/apps/{dotnet,laravel,wordpress}/` will land in **Other** until its name prefix is added here (or a future `health.group` label exists).

**Latest verified check (2026-10-10):** 20 containers, 7 certs. See [`../stella-server/health-api.md`](../stella-server/health-api.md).

### Backup status (from health-api)

Possible `backup.status` values: `ok`, `warning`, `failed`, `stale`, `unknown`. Atlas treats `warning` / `stale` as degraded and `failed` as down (per the aggregate rules below; `failed` is in the down set).

The dashboard backup tile matches the other tiles: colour dot + `30/30` (last 30 `history.jsonl` entries). `/monitoring/stella` has the full details: status, duration, snapshot size / count, weekly verify, dumps, `problems`, and a size column on the history table.

**UNVERIFIED:** whether a failed backup sends an email via Brevo. The docs only describe emails for WP site `down` events. **TODO (Dominik):** check whether the fake-failure test produced an email; if not, decide on an Atlas-side alert.

### Aggregate status thresholds (Atlas side)

`StellaHealthRecorder` maps service statuses to an overall check status:

| Check status | When |
|---|---|
| `down` | Any service has status `down`, `missing`, or `error` |
| `degraded` | Any service has status `degraded`, `warning`, `incomplete`, or `unconfigured` |
| `ok` | All services `up` or `ok` |

### CPU / load warnings (Atlas side, from health-api payload)

| Metric | Warning threshold | Down threshold |
|---|---|---|
| `load_1m` (host, 12-thread) | ≥ 8 | ≥ 10 |
| `ram_used_pct` | ≥ 85 % | ≥ 95 % |
| `disk used_pct` | ≥ 85 % | ≥ 95 % |
| Ollama `cpu_pct` | ≥ 550 % (warning) | — |

Ollama CPU is reported as sum across all cores (6 capped cores → 600 % max); ≥ 550 % is "near ceiling."

### Probe fallback (when health-api is not configured / unreachable)

`StellaHealthService` fires parallel GET requests to configured public endpoints:

| Service | Env var | Probe URL |
|---|---|---|
| Deploy API | `STELLA_DEPLOY_API_URL` | `/health` |
| IMAP Sync | `STELLA_IMAP_SYNC_URL` | `/health` |
| Stella API | `STELLA_API_URL` | `/chat/health` |
| Ollama | `STELLA_OLLAMA_URL` | `/api/version` |
| Caddy | `STELLA_CADDY_URL` | `/` |
| Advoapp (dev) | `STELLA_ADVOAPP_URL` | `/` |
| Osgar Datahub | `STELLA_OSGAR_URL` | `/` |

An unconfigured env var produces `status: unconfigured` (not `down`); missing env vars don't fail the check.

### API endpoints

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/api/monitoring/stella` | Latest `StellaHealthCheck` with all `serviceLogs` |
| `POST` | `/api/monitoring/stella/check` | Trigger an on-demand check (same as the cron job) |
| `GET` | `/api/monitoring/stella/metrics` | Recent check history for time-series charts |

### Required env vars

```
STELLA_HEALTH_API_URL=https://stella-health-api.foxcraft.digital   # primary (signed)
# Optional probes (used as fallback or in addition to health-api):
STELLA_DEPLOY_API_URL=https://stella-deployment-api.foxcraft.digital
STELLA_IMAP_SYNC_URL=https://stella.foxcraft.digital/imap-sync
STELLA_API_URL=https://stella.foxcraft.digital/stella
STELLA_OLLAMA_URL=https://stella.foxcraft.digital/ollama
STELLA_CADDY_URL=https://stella.foxcraft.digital
STELLA_ADVOAPP_URL=https://advoapp.finditoo.foxcraft.digital
STELLA_OSGAR_URL=https://osgar.datahub.foxcraft.digital
```

**Stella-side health-api docs:** [`../stella-server/health-api.md`](../stella-server/health-api.md)

---

## Atlas host monitoring

`CheckAtlasHealth` runs every 5 minutes. It signs `system-health:{timestamp}` and POSTs to `ATLAS_HOST_API_URL` (`https://api.atlas.foxcraft.digital/system/health` by default). Timeout 60s. No public-probe fallback — a failed call is stored as `source=host-api-error`.

Hairpin from Atlas itself to the public hostname may fail. Use `ATLAS_HOST_API_URL=http://127.0.0.1:8090` on the Atlas box if that happens. See [`host-api.md`](host-api.md).

### Data model

```
AtlasHealthCheck
  id, status (ok|degraded|down), source, response_ms, payload (JSON), error, checked_at

AtlasServiceUptimeLog
  id, atlas_health_check_id (FK cascade), service_id, service_label, category,
  status, response_ms, message, meta (JSON), checked_at
```

**Retention:** `ATLAS_HEALTH_RETENTION_DAYS` (default 30).

The UI is the pink **Server: atlas** card on the dashboard and `/monitoring/atlas`. CPU% is `load_1m / ATLAS_HOST_CPU_THREADS` (default 4, CX33).

### API endpoints

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/api/monitoring/atlas` | Latest `AtlasHealthCheck` with service logs |
| `POST` | `/api/monitoring/atlas/check` | On-demand check |
| `GET` | `/api/monitoring/atlas/metrics` | Recent check history for charts |
| `POST` | `/api/monitoring/atlas/deploy` | Start a production deploy (atlas.dev only; 404 when the flag is off) |
| `GET` | `/api/monitoring/atlas/deploy/status` | Poll deploy status + `log_tail` (atlas.dev only; 404 when the flag is off) |
| `GET` | `/api/monitoring/hetzner` | Live Hetzner Cloud snapshot (atlas + edison + extras, cached) |
| `POST` | `/api/monitoring/hetzner/refresh` | Drop the cache and refetch |

`ATLAS_SELF_DEPLOY_ENABLED` must stay `false` on production. When true (atlas.dev), the Server: atlas card shows **Deploy production**. The Laravel client signs `deploy-atlas` / `deploy-status` against `ATLAS_HOST_API_URL`. The host API has no SSE; the UI polls status. See [`host-api.md`](host-api.md).

```
ATLAS_HOST_API_URL=https://api.atlas.foxcraft.digital
ATLAS_HOST_API_TIMEOUT=60
ATLAS_HOST_CPU_THREADS=4
ATLAS_HEALTH_RETENTION_DAYS=30
ATLAS_SELF_DEPLOY_ENABLED=false
```

---

## Hetzner Cloud (Atlas + Edison)

Atlas and Edison are CX33s in the **same** Hetzner Cloud project. Atlas reads that project through `api.hetzner.cloud` (read-only token) and shows server facts plus automatic Cloud backups.

This is a different layer from Stella’s restic → Storage Box backups and from the optional Atlas host-api `backup` object.

`GET /api/monitoring/hetzner` (cached 120s). `POST /api/monitoring/hetzner/refresh` drops the cache. The token is never returned to the browser.

| Field | Source |
|---|---|
| status, type, cores, RAM, disk, location, IPv4 | `GET /servers` |
| backup window / enabled | `server.backup_window` (`null` = backups off) |
| Cloud backup images | `GET /images?type=backup` (`bound_to` = server id) |
| Snapshots | `GET /images?type=snapshot` (`created_from.id`) |
| Volumes | `GET /volumes` |

Servers are matched by name (`atlas`, `edison`), then by IPv4 (`2.31.42.13`, `178.105.203.54`). Extra servers in the project are listed without a role.

| `backup_status` | When |
|---|---|
| `ok` | Backups enabled, newest image ≤ 36 h |
| `degraded` | Newest image 36–72 h |
| `down` | Enabled but no image, or newest > 72 h |
| `unconfigured` | `backup_window` is null |

Token (read-only) lives in **Settings → Hetzner Cloud** (`app_settings.hetzner_cloud_token`, encrypted). `HETZNER_CLOUD_TOKEN` in `.env` is a fallback only. The token is never returned to the browser.

```
HETZNER_CLOUD_ATLAS_NAME=atlas
HETZNER_CLOUD_ATLAS_IPV4=2.31.42.13
HETZNER_CLOUD_EDISON_NAME=edison
HETZNER_CLOUD_EDISON_IPV4=178.105.203.54
```

UI: Settings → Hetzner Cloud (save + test); Atlas detail page **Hetzner Cloud** section; dashboard **Server: edison** card + `/monitoring/edison`. Edison has no host API yet — Cloud is the only live signal.

---

## WP Sites uptime monitoring

### Data model

```
WpSite
  id, name, domain, environment (production|staging|development),
  secret_key (encrypted), wp_user,
  uptime_status (up|degraded|down|null), uptime_response_ms,
  last_backup_at, last_backup_type, ssl_expires_at, plugin_updates_count,
  last_checked_at

WpSiteUptimeLog
  id, wp_site_id (FK), status, response_ms, checked_at
```

**`has_issues`** (computed attribute) flags a site when:
- `uptime_status` is `down` or `degraded`, **or**
- SSL cert expires in ≤ 14 days, **or**
- `plugin_updates_count > 0`, **or**
- `last_backup_type === 'failed'`

**Log retention:** 7 days (pruned after each check, per site).

### How uptime is checked

`CheckWpSiteUptime` performs a `curl HEAD https://{domain}` with:
- 10 s timeout, up to 3 redirects followed, SSL verification **disabled** (avoids false positives from slightly-expired certs when the real question is "is the site up?")
- `status: up` if HTTP 2xx/3xx and response < 1 000 ms
- `status: degraded` if HTTP 2xx/3xx and response ≥ 1 000 ms
- `status: down` if curl fails or HTTP code 0

### Down alerts

When a site transitions to `down` (previous status was not `down`), Atlas sends a transactional email via **Brevo**. A recovery alert is not sent — only the first `down` event triggers a notification per outage.

Alert email is sent to `brevo_alert_email` (Settings → Email).

### API endpoints

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/api/wp-sites` | List all sites with latest status |
| `POST` | `/api/wp-sites` | Register a new site |
| `PATCH` | `/api/wp-sites/{id}` | Update site config |
| `DELETE` | `/api/wp-sites/{id}` | Remove a site |
| `POST` | `/api/wp-sites/{id}/login` | Generate WP auto-login URL (requires `secret_key` + `wp_user`) |
| `POST` | `/api/wp-sites/{id}/refresh` | Trigger an immediate uptime check for one site |

---

## Settings required for alerts

Both monitoring features can send email alerts. Configured in **Settings → Email** (stored in `app_settings` table):

| Key | Purpose |
|---|---|
| `brevo_api_key` | Brevo transactional email API key |
| `brevo_from_email` | Sender email address |
| `brevo_from_name` | Sender display name (defaults to app name) |
| `brevo_alert_email` | Recipient for WP site down alerts. **UNVERIFIED:** whether a failed Stella backup also emails this address. **TODO (Dominik):** check the fake-failure test. |

If `brevo_api_key` is not set, alerts are silently skipped — no exception, just a log warning.

---

## Further reading

- Atlas Connect login and backups (not part of the uptime loop): [`atlas-connect/`](atlas-connect/)
- Stella-side health-api: [`../stella-server/health-api.md`](../stella-server/health-api.md)
- Stella backup + restore: [`../stella-server/backup.md`](../stella-server/backup.md), [`../stella-server/restore.md`](../stella-server/restore.md)
- System map: [`../integration/system-overview.md`](../integration/system-overview.md)
