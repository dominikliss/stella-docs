# Atlas stacks

**`/stacks`** is the operator UI for Stella compose stacks. Atlas does not talk to Docker itself: every call is a signed request to [`../stella-server/stack-api.md`](../stella-server/stack-api.md). The browser never calls Stella.

**Status (2026-11-13):** live on **atlas.dev**. Prod flag still off. First stack created end to end from Atlas on 2026-11-08: `sandbox.staging.foxcraft.digital`.

| Gate | Effect |
|------|--------|
| `STELLA_STACK_API_ENABLED` | Off → `/api/stacks*` is 404 and the nav item is hidden (`me.features.stack_api`) |
| Admin role | Non-admin → 403 even when the flag is on |
| Principal scope | `atlas-dev` sees only `STACK_API_SANDBOX_STACKS` (`dominikliss.foxcraft.digital`, `sandbox.staging.foxcraft.digital`). `atlas-prod` sees all `/opt/apps/{wordpress,laravel,dotnet}/*` stacks |

Token stays in `.env` (`STELLA_STACK_API_TOKEN`). Signing key: Settings → GitHub `stack_api_ssh_key_path` (fallback `STELLA_STACK_API_KEY_PATH`). atlas.dev uses `/var/www/.ssh/id_ed25519` (principal `atlas-dev`). Prod key when enabled: `/home/atlas/.ssh/atlas-stack-api` (principal `atlas-prod`). Namespace **`stack`**, not `deploy`.

---

## What it can do

| Capability | Live |
|------------|------|
| List stacks (type, sandbox tag, container count, all-running / N-stopped) | yes |
| Search by name or service; filter All / Issues; type tabs when more than one type is present | yes |
| Detail: containers (image, state, status, started, restarts, health, bind ports) | yes |
| Service logs (50 / 200 / 1000 lines, newest at bottom, truncated warning) | yes |
| Restart one service or the whole stack (sync or async job) | yes |
| Fix WordPress `html/` permissions (queued job) | yes |
| Plan a new WordPress stack (dry-run compose / files / Caddy) | yes |
| Create from a matching plan (queued apply job; URL + SSH port on success) | yes |
| People list for SSH users (names + uids, no keys) | yes |
| Recent stack-api jobs + Atlas `stack_actions` history | yes |
| Update an existing stack | **not built** |
| Delete / trash | **not built** (sandbox.staging can only be removed by hand) |
| Restore from an Atlas backup | **not built** (`create` + `restore` is the planned replacement for wp-staging-api) |
| Create Laravel or .NET stacks | **not built** (list/get still show those types) |

List polls every 30 s. Detail polls every 30 s; jobs every 10 s; a watched job every 2 s until `ok` / `error`.

---

## List (`/stacks`)

Sorted WordPress → Laravel → .NET → anything else, then name. A **sandbox** tag marks names in `STACK_API_SANDBOX_STACKS`. Status is derived from container `state`: all `running` → `all running`; some stopped → `N stopped`. Open bind IPs (`0.0.0.0` / `::`) are highlighted on the detail page, not the list.

**New stack** opens the plan drawer. Create is WordPress-only in the UI.

---

## Create (Plan → Create)

The form is a dry-run plus an apply job. Plan writes nothing. Create is disabled until a **successful** plan matches the current form (changing name, PHP, SSH users, or ini rows requires Plan again).

| Field | Rule |
|-------|------|
| Name | One DNS label; Atlas appends `.staging.foxcraft.digital`. Pattern `^[a-z0-9]([a-z0-9-]{0,61}[a-z0-9])?$`. Reserved: `atlas`, `api`, `stella`, `edison`, `york`, `www`, `mail`, `stella-*` |
| Type | WordPress (locked) |
| PHP | `8.5` (locked; stack-api allowlist) |
| SSH users | At least one name from `GET /api/stack-people`. Defaults to every returned person. uid + key come from Stella `people/`, not the form |
| PHP settings | Optional. Whitelist only: `memory_limit`, `upload_max_filesize`, `post_max_size`, `max_execution_time`, `max_input_time`, `max_input_vars`, `display_errors`, `date.timezone`, `session.gc_maxlifetime`, `opcache.enable`, `opcache.memory_consumption`, `opcache.max_accelerated_files`, `opcache.validate_timestamps`, `opcache.revalidate_freq` |

Plan response (always HTTP 200 when handled): path, SSH port (or “no SSH”), SSH users, notes, and artifacts (`docker-compose.yml`, extra files, Caddy site block). `ok: false` with a rendered compose means policy failed (e.g. a disallowed `php_ini` key). Existing stacks / folders / Caddy files are rejected (`update is not supported yet`). A sandbox principal may only plan/apply names in `STACK_API_SANDBOX_STACKS`.

Create confirms, then `POST /api/stack-apply`. Atlas records `stack_actions` as `apply` / `queued` with the job id. Success result: public URL (`https://<name>`), SSH port in **22100–22199**, SSH users, Open stack. Failure result: `detail`, failed `step`, output, done / rollback lists. Apply is a heavy job (one at a time on Stella).

New sites are IP-restricted in Caddy (`STACK_API_SITE_ALLOW_IPS`: Proton, york, Atlas prod). Wildcard cert for `*.staging.foxcraft.digital` — the site block has no `tls` section.

---

## Detail (`/stacks/$name`)

| Action | Notes |
|--------|-------|
| Logs | Per service. Docker stdout/stderr, timestamps on. 50 / 200 / 1000 lines (API 1–1000, default 200). Panel scrolls to the newest line. Truncated containers are marked. Log bodies are never written to the Stella audit log |
| Restart service | Confirm dialog. Optional **Async** → `{job, state:"queued"}` and Atlas watches the job |
| Restart stack | Same, plus type-the-name confirm unless the stack is sandbox. Stella restarts `db` first |
| Fix permissions | WordPress only. Queued job. `chown` www-data, `g+rwX`, SetGID, default ACL on `html/` so SSH users in that group can write. `400` `"only wordpress stacks"` / `"stack folder not available"`; `409` stack busy |

Busy stack (sync restart or any job already running) → `409` “Stack is busy — another action is running.”

**Recent jobs** come from stack-api (`limit` 10 on the page). **History** is Atlas `stack_actions` (last 50): restart, apply, fix-permissions. Queued rows store the job id in `detail` and settle to `ok` / `error` when the job is fetched (detail page, job list, or job get). `plan` and `people` do not write `stack_actions`. Stella keeps its own append-only audit log.

---

## Atlas API

Admin only. 404 when the flag is off. Client: `app/Domain/Stacks/StackApiClient.php`. Timeouts: restart 65 s (15 s when async), logs 30 s, plan 30 s, apply 30 s, people / fix-permissions 15 s. Every Stella body includes a random `nonce`. Empty `php_ini` is `{}`, never `[]`.

| Atlas | Stella action |
|-------|---------------|
| `GET /api/stacks` | `stack-list` |
| `GET /api/stacks/{name}` | `stack-get` (+ local history) |
| `GET /api/stacks/{name}/logs` | `stack-logs` |
| `POST /api/stacks/{name}/restart` | `stack-restart` |
| `POST /api/stacks/{name}/fix-permissions` | `stack-fix-permissions` |
| `GET /api/stack-jobs` / `{id}` | `job-list` / `job-get` |
| `GET /api/stack-people` | `people-list` |
| `POST /api/stack-plan` | `stack-plan` |
| `POST /api/stack-apply` | `stack-apply` |

Use `/api/stack-jobs`, `/api/stack-plan`, `/api/stack-people`, `/api/stack-apply` — not `/api/stacks/jobs` and so on (`jobs` would match `{name}`).

Name `^[a-z0-9][a-z0-9.-]{0,252}$`, service `^[a-z0-9][a-z0-9_-]{0,62}$`, job id `^[0-9a-f]{32}$` — validated in Atlas before the call. Plan names are not pre-validated so Stella can return sandbox / reserved-name errors.

Mapped errors: 401/403 → `stack-api rejected the request`; 400/404 detail passed through; 409 → stack busy; 502 → docker error (results passed through); connection → `stack-api unreachable`. A handled plan or apply rejection is HTTP 200 with `ok` + `errors`.

Never log the token, signature, key contents, log lines, spec keys, or response files.

---

## Not in this UI

Stella still discovers Laravel and .NET stacks for list/get/restart/logs. Create has no blueprint for them yet. People changes in `stack-api/people/` need a rebuild of the API image and do not update running stacks. Roadmap (update, delete, restore, other types): [`../stella-server/stack-api.md#roadmap`](../stella-server/stack-api.md#roadmap).
