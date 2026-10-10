# Atlas host API

Signed FastAPI on the Atlas production host. Health snapshot of the box, plus a deploy trigger that only Atlas **dev** (on Stella) may call.

**Live at:** `https://api.atlas.foxcraft.digital` → `atlas-api` on `127.0.0.1:8090`  
**Host runbook:** [`server.md`](server.md). Atlas-side polling: [`monitoring.md`](monitoring.md). Same SSH-signature scheme as Stella health-api: [`../stella-server/health-api.md`](../stella-server/health-api.md). Map: [`../integration/system-overview.md`](../integration/system-overview.md).

The deploy function must **not** appear in the Atlas production app. Only Atlas dev (on Stella) may trigger a deploy of production.

---

## Layout

```
/opt/atlas-api/                    # FastAPI, systemd atlas-api, user atlasapi
├── auth.py
├── deploy.py                      # starts and tracks the deploy script
├── allowed_signers                # atlas-prod + atlas-dev
└── allowed_signers_deploy         # atlas-dev only; root:atlasapi 640

/var/lib/atlas-api/                # owner atlasapi
├── deploy.json
└── deploy.log
```

| Path | Role |
|------|------|
| `auth.py` | `verify_signature(payload, signature, signers_path)` and `verify_request(action, req, signers_path=ALLOWED_SIGNERS_PATH)`. Also `DEPLOY_SIGNERS_PATH`. |
| `deploy.py` | Starts `/usr/local/bin/atlas-deploy` detached and tracks status. Returns immediately — the build takes minutes and restarts services. |
| `allowed_signers` | `atlas-prod` + `atlas-dev`. Used for `system-health`. |
| `allowed_signers_deploy` | Only the `atlas-dev` key, one `ddashboard` line. Used for `deploy-atlas` and `deploy-status`. |
| `/var/lib/atlas-api/deploy.json` | Last deploy state (id, ref, status, pid, timestamps, exit code). |
| `/var/lib/atlas-api/deploy.log` | stdout/stderr of the running / last deploy. |

Other modules in `/opt/atlas-api/` (the FastAPI app itself) are **TODO (Dominik):** list filenames.

---

## Endpoints

| Method | Path | Auth | Purpose |
|--------|------|------|---------|
| GET | `/health` | none | Liveness. Returns `{"status":"ok"}`. |
| POST | `/system/health` | signed (`system-health`), `allowed_signers` | Host snapshot (systemd units, certs, disk/CPU/RAM, optional backup). Atlas `CheckAtlasHealth` polls this. |
| POST | `/system/deploy` | signed (`deploy-atlas`), `atlas-dev` key only | Starts `/usr/local/bin/atlas-deploy` detached. Payload `deploy-atlas:<unix_ts>` (ref defaults to `main`) or `deploy-atlas:<ref>:<unix_ts>`. Returns `{"id","ref","status":"running"}`. `409` if a deploy is already running, `400` for an invalid ref (regex `^[A-Za-z0-9._][A-Za-z0-9._/-]*$`, no `..`) |
| POST | `/system/deploy/status` | signed (`deploy-status`), `atlas-dev` key only | Returns `id`, `ref`, `status` (`running` / `success` / `failed` / `unknown` / `none`), `pid`, `started_at`, `finished_at`, `exit_code`, `log_tail` (last 60 lines) |

---

## Auth

Two signers files. Same scheme as Stella (`ssh-keygen -Y sign` / `-Y verify`, namespace `deploy`, principal `ddashboard`). See [`../stella-server/deploy-api.md`](../stella-server/deploy-api.md#auth-mechanism).

| File | Keys | Used for |
|------|------|----------|
| `allowed_signers` | `atlas-prod` + `atlas-dev` | `system-health` |
| `allowed_signers_deploy` | `atlas-dev` only (one `ddashboard` line) | `deploy-atlas`, `deploy-status` |

The prod key on a deploy endpoint is rejected with `401 signature verification failed`.

**Decision:** only Atlas dev (on Stella) may trigger a deploy of production. The deploy function must not appear in the Atlas production app.

Signing a health request (from Atlas prod or dev) is the same pattern as Stella health-api: payload `system-health:<unix_ts>`, `-n deploy`.

Signing a deploy from Stella (Atlas dev):

```bash
ssh-keygen -Y sign -f /opt/apps/laravel/atlas.dev.foxcraft.digital/deploy-key/id_ed25519 -n deploy dp.txt
```

Payload `deploy-atlas:main:<ts>`, POST to `https://api.atlas.foxcraft.digital/system/deploy`.

---

## IP whitelist

Hetzner Cloud firewall already allows TCP/443 from `95.217.144.93` (Stella) and from `194.126.177.181`. Caddy is the second layer. In the `api.atlas` Caddy block:

```
@deploy_blocked { path /system/deploy* ; not remote_ip 95.217.144.93 }
respond @deploy_blocked 403
```

Complete current `api.atlas` block:

```caddyfile
api.atlas.foxcraft.digital {
    tls {
        issuer acme {
            email projects@foxcraft.digital
            dir https://acme-v02.api.letsencrypt.org/directory
            dns hetzner {env.HETZNER_API_TOKEN}
            propagation_delay 90s
            propagation_timeout 5m
        }
    }
    @blocked {
        not remote_ip 2.31.42.13 95.217.144.93
    }
    respond @blocked 403
    @deploy_blocked {
        path /system/deploy*
        not remote_ip 95.217.144.93
    }
    respond @deploy_blocked 403
    reverse_proxy 127.0.0.1:8090
}
```

After `rm` + recreate of `/etc/caddy/Caddyfile`, `chmod 644 /etc/caddy/Caddyfile` is required (`atlasapi` reads it).

---

## Deploy endpoint

The script `/usr/local/bin/atlas-deploy` already existed (`root:root`, mode 750). The API only calls it as `sudo -n /usr/local/bin/atlas-deploy <ref>`, detached (`start_new_session`), stdout and stderr go to `/var/lib/atlas-api/deploy.log`, the exit code to `deploy.json`.

Sudoers: `/etc/sudoers.d/atlas-api` (mode 440) with exactly:

```
atlasapi ALL=(root) NOPASSWD: /usr/local/bin/atlas-deploy
```

What the script does: validates the ref, takes `flock` on `/var/lock/atlas-deploy.lock`, `git fetch --tags`, `git checkout <ref>`, `git pull --ff-only` when on a branch, exits early with `already at <sha>` if the commit is unchanged, otherwise as user `atlas`: `composer install --no-dev --optimize-autoloader`, `php artisan migrate --force`, `php artisan config:clear`, `npm ci --legacy-peer-deps && npm run build`, then as root `systemctl reload php8.5-fpm` and `systemctl restart atlas-queue`. Prints `deployed <old> -> <new> (<ref>)`.

Two layers against parallel runs: the script's `flock` and the API's `409`.

`deploy.py` returns immediately because the build takes minutes and restarts services.

Manual fallback (same script, run on the host): [`server.md`](server.md#deploy-code).

---

## Verified

**2026-10-10 (health):** host API live at `https://api.atlas.foxcraft.digital`. Atlas-side client `CheckAtlasHealth` stores `atlas_health_checks` / `atlas_service_uptime_logs` and renders the dashboard card + `/monitoring/atlas`. See [`monitoring.md`](monitoring.md).

**2026-10-11 (Atlas dev deploy client):** `ATLAS_SELF_DEPLOY_ENABLED` gates `POST /api/monitoring/atlas/deploy` and `GET /api/monitoring/atlas/deploy/status`. Off on prod (404). On atlas.dev the Server: atlas card shows a Deploy production button that signs `deploy-atlas` / `deploy-status` and toasts success or failure. No SSE — the UI polls `log_tail`.

**2026-10-10 (deploy):**

- Dev key from Stella: `POST /system/deploy` accepted, status `success`, exit code 0, log `already at b087616`, about 2.5 s.
- Prod key on `127.0.0.1:8090/system/deploy`: `{"detail":"signature verification failed"}`.
- Public URL from atlas itself: `403` (Caddy path matcher).
- **Not yet verified:** a deploy that pulls new commits (composer, migrate, npm build, reload, queue restart) started through the API.

---

## Gotchas / risks

1. `systemctl restart atlas-queue` at the end of the script kills a running Atlas customer deploy job (`DeployRunJob`, timeout up to 2 hours). Graceful alternative: `sudo -u atlas php artisan queue:restart`. Script not changed yet. **TODO (Dominik)**
2. The script's ref regex `^[A-Za-z0-9._/-]+$` allows a leading `-`, which `git checkout` can read as an option. The API rejects it, the script does not. **TODO (Dominik)**
3. Whoever controls the dev instance or its `deploy-key` can deploy any ref of the atlas repo to production. The dev instance is the weaker environment (`APP_ENV=local`, `APP_DEBUG=true`). Consider restricting deploys to tags later.
4. `/tmp/*.txt` and `*.sig` files left over from root test runs block user `atlas` from overwriting them (permission denied); delete as root before re-testing.
5. Caddy ACME staging fallback on `api.atlas` was caused by missing `propagation_delay` (NXDOMAIN on `_acme-challenge` TXT during secondary validation). Fix: `propagation_delay 90s` + `propagation_timeout 5m` in the `issuer acme` block, then stop/start caddy.
6. Hairpin from Atlas itself to `https://api.atlas.foxcraft.digital` may fail. Use `ATLAS_HOST_API_URL=http://127.0.0.1:8090` on the Atlas box if that happens. **TODO (Dominik):** confirm which URL production `CheckAtlasHealth` actually uses.

---

## Open items

- Atlas prod via the public API URL: test `https://api.atlas.foxcraft.digital` from Atlas itself (hairpin risk), otherwise use `127.0.0.1:8090` locally.
- First real deploy through the host API (new commits) — record the result in the Verified list above.
- Patch `/usr/local/bin/atlas-deploy`: `queue:restart` instead of `systemctl restart atlas-queue`; reject refs starting with `-`.
