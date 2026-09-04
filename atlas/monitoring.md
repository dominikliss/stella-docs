# Atlas Monitoring

Atlas monitors two things on a 5-minute cron loop:

1. **Stella server health** — signed requests to Stella's health-api; full container/TLS/disk/CPU/RAM/Ollama status
2. **WP site uptime** — plain HTTPS HEAD ping to each registered WordPress site

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

| Category | Services |
|---|---|
| `system` | `system:docker_daemon`, `system:ollama_models`, `system:ollama_resources`, `system:cpu_ram`, `system:disk` |
| `container` | `container:stella-api`, `container:deploy-api`, `container:health-api`, `container:caddy`, `container:imap-sync`, `container:ollama`, `container:backup` |
| `app` | Any container not in the infra list (e.g. `container:advoapp-dev`, `container:osgar-datahub-dev`) |
| `http` | `http:stella-api`, `http:ollama` |
| `cert` | `cert:stella.foxcraft.digital`, `cert:advoapp.finditoo.foxcraft.digital`, etc. |

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
| `brevo_alert_email` | Recipient for WP site down alerts |

If `brevo_api_key` is not set, alerts are silently skipped — no exception, just a log warning.
