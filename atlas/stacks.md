# Atlas stacks

**`/stacks`** is the operator UI for Stella compose stacks. Atlas does not talk to Docker itself: every call is a signed request to [`../stella-server/stack-api.md`](../stella-server/stack-api.md). The browser never calls Stella.

**Status (2026-11-14):** live on **atlas.dev**. Prod flag still off. First stack created end to end from Atlas on 2026-11-08: `sandbox.staging.foxcraft.digital` (adopted into Stella `stack_state`, owner `atlas-dev`).

| Gate | Effect |
|------|--------|
| `STELLA_STACK_API_ENABLED` | Off → `/api/stacks*` is 404 and the nav item is hidden (`me.features.stack_api`) |
| Admin role | Non-admin → 403 even when the flag is on |
| Principal scope | `atlas-dev` (sandbox) lists stacks it can see, including any `<label>.staging.foxcraft.digital` name it created (it then owns that name). `atlas-prod` sees all `/opt/apps/{wordpress,laravel,dotnet}/*` stacks |

Token stays in `.env` (`STELLA_STACK_API_TOKEN`). Signing key: Settings → GitHub `stack_api_ssh_key_path` (fallback `STELLA_STACK_API_KEY_PATH`). atlas.dev uses `/var/www/.ssh/id_ed25519` (principal `atlas-dev`). Prod key when enabled: `/home/atlas/.ssh/atlas-stack-api` (principal `atlas-prod`). Namespace **`stack`**, not `deploy`.

---

## What it can do

| Capability | Live |
|------------|------|
| List stacks (type, sandbox tag, managed badge, drift badge, container count, all-running / N-stopped) | yes |
| Search by name or service; filter All / Issues; cards grouped by WordPress / Laravel / .NET | yes |
| Detail: container cards (image, state, status, started, restarts, health, bind ports) | yes |
| Service logs (50 / 200 / 1000 lines, newest at bottom, truncated warning) | yes |
| Restart one service or the whole stack (sync or async job) | yes |
| Fix WordPress `html/` permissions | apply still runs this; Atlas UI does not expose the recovery action |
| Plan a new WordPress stack (dry-run compose / files / Caddy; mode `create`) | yes |
| Create from a matching plan (queued apply job; URL + SSH port on success) | yes |
| WordPress login after create (URL, user, email, password once from the job result, table prefix) | yes |
| Edit a managed WordPress stack (plan diffs → apply; job action `stack-update`) | yes |
| Reconcile drift (`adopt-live` sync or `re-render` job) | yes |
| People list + out-of-sync stacks; sync one or all | yes |
| Recent stack-api jobs + Atlas `stack_actions` history | yes |
| Delete a managed WordPress stack (queued job; trash for `retention_days`) | yes |
| Trash list + restore (queued job) | yes |
| Restore from an Atlas backup | **not built** (`create` + `restore` is the planned replacement for wp-staging-api) |
| Create Laravel or .NET stacks | **not built** (list/get still show those types) |
| PHP version change | **not built** |

List polls every 30 s. Detail polls every 30 s; jobs every 10 s; a watched job every 2 s until `ok` / `error`.

---

## List (`/stacks`)

Card grid grouped by WordPress → Laravel → .NET → anything else, then name. Each card has a scaled live preview when the stack name is a hostname (`https://{name}` in a sandboxed iframe), a status dot, the name, a **drift** badge when `drift` is true, and a restart control. Clicking a card opens the stack in a sidebar (URL `/stacks/$name`) instead of a separate page. Container chips and sandbox/managed pills stay off the card. Status is derived from container `state` (dot: all running / degraded / down). Open bind IPs (`0.0.0.0` / `::`) are highlighted in the sidebar, not the list.

**Stacks / People / Trash** tabs. **New stack** opens the plan drawer. Create is WordPress-only in the UI.

---

## Create (Plan → Create)

The form is a dry-run plus an apply job. Plan writes nothing. Create is disabled until a **successful** plan matches the current form (changing name, PHP, SSH users, or ini rows requires Plan again).

| Field | Rule |
|-------|------|
| Name | One DNS label; Atlas appends `.staging.foxcraft.digital`. Pattern `^[a-z0-9]([a-z0-9-]{0,61}[a-z0-9])?$`. Reserved: `atlas`, `api`, `stella`, `edison`, `york`, `www`, `mail`, `stella-*` |
| Type | WordPress (locked) |
| PHP | `8.4` or `8.5` (default `8.5`). Locked on edit — Stella changes PHP with the PHP version job, not with update |
| SSH users | At least one name from `GET /api/stack-people`. Defaults to every returned person. uid + key come from Stella `people/`, not the form |
| PHP settings | Always shown with blueprint defaults and a value dropdown per key. Overlay from `spec.php_ini` on edit. Keys: `memory_limit` `256M`, `upload_max_filesize` `64M`, `post_max_size` `64M`, `max_execution_time` `300`, `max_input_time` `60`, `max_input_vars` `1000`, `display_errors` `Off`, `date.timezone` `Europe/Vienna`, `session.gc_maxlifetime` `1440`, `opcache.enable` `1`, `opcache.memory_consumption` `128`, `opcache.max_accelerated_files` `10000`, `opcache.validate_timestamps` `0`, `opcache.revalidate_freq` `2` |

Plan response (always HTTP 200 when handled) includes `mode: "create"`: path, SSH port (or “no SSH”), SSH users, notes, and artifacts (`docker-compose.yml`, extra files, Caddy site block). `ok: false` with a rendered compose means policy failed (e.g. a disallowed `php_ini` key). An existing **managed** stack is an update (below), not a create. Unmanaged existing stacks / folders / Caddy files are still refused. A name in the trash is rejected (`name: <name> is in the trash until <ts>; restore it or wait for the purge`). A sandbox principal may plan/apply any valid `<label>.staging.foxcraft.digital` name; after apply it owns that stack and it shows up in its list.

Create confirms, then `POST /api/stack-apply`. Response includes `mode: "create"`. Atlas records `stack_actions` as `apply` / `queued` with the job id. Success result: public URL (`https://<name>`), SSH port in **22100–22199**, SSH users, Open stack, and a **WordPress login** block when `result.admin` is present: login URL `{result.url}/wp-admin/`, user, email, table prefix, and the admin password (masked, reveal + copy). If `admin.password` is `null`, the password row is hidden and the block says “already installed”. The password is shown only from the stack-api job result; Atlas does not write it to `stack_actions`, history, or any Atlas table, and does not log it. Failure result: `detail`, failed `step`, output, done / rollback lists. Apply is a heavy job (one at a time on Stella).

New sites are IP-restricted in Caddy (`STACK_API_SITE_ALLOW_IPS`: Proton, york, Atlas prod). Wildcard cert for `*.staging.foxcraft.digital` — the site block has no `tls` section.

---

## Detail (sidebar on `/stacks/$name`)

| Action | Notes |
|--------|-------|
| Logs | Per service. Docker stdout/stderr, timestamps on. 50 / 200 / 1000 lines (API 1–1000, default 200). Panel scrolls to the newest line. Truncated containers are marked. Log bodies are never written to the Stella audit log |
| Restart service | Confirm dialog. Always queued (`async`); Atlas watches the job |
| Restart stack | On the card and in the sidebar. Always queued. Type-the-name confirm unless the stack is sandbox. Stella restarts `db` first |
| Edit | Shown only when `managed` is true. Opens the plan drawer prefilled from `spec`. Disabled when `drift` is true |
| Reconcile | Shown when `drift` is true. Dialog: adopt live files, or re-render from the stored spec. Re-render on a non-sandbox stack needs a type-the-name confirm |
| Delete | Shown only when `managed` is true. Type-the-name confirm. Stack goes to the trash for `retention_days` and can be restored. Queued job; on success toast + close the sidebar. On failure: failed `step` + output. Atlas always sends `final_dump: false` (no SQL copy; restore uses volumes) |

`drift` is `true` / `false` / `null` (`null` = not managed). A **drift** badge is on the card and in the sidebar when it is true.

Busy stack (sync restart or any job already running) → `409` “Stack is busy — another action is running.” Delete also maps 409 `stack is not managed by stack-api` / `stack is already in the trash` / `stack busy` (message stays `stack busy`; detail is passed through). Adopt-live 409 `live files do not match a blueprint render; use re-render` is shown in the reconcile dialog (message still `stack busy`; detail is passed through).

**Recent jobs** come from stack-api (`limit` 10 on the page). **History** is Atlas `stack_actions` (last 50): restart, apply, update, reconcile, people-sync, fix-permissions, delete, trash-restore. Queued rows store the job id in `detail` and settle to `ok` / `error` when the job is fetched (detail page, job list, or job get). `plan`, `people`, and trash list do not write `stack_actions`. Stella keeps its own append-only audit log.

---

## Edit (Plan → Apply)

Managed stacks only. The drawer is the same form as create, prefilled from `GET /api/stacks/{name}` `spec`. **Name** and **type** are locked. **PHP** is locked with the hint “change via PHP version (coming)” — Stella rejects a PHP change on update (`php: is 8.5; the PHP version is changed with the PHP version job, not with update`). Apply is disabled until a **successful** plan matches the current form (SSH users or ini rows).

Plan for an existing managed stack returns `mode: "update"` plus `drift`, `changes: [{file, status: added|changed|removed, diff}]`, notes, and the usual artifacts. Atlas shows the notes and a collapsible unified diff per changed file (monospace, `+` / `-` colored). `ok: false` with `drift: live files differ from the last apply; reconcile first` means Edit is the wrong action — reconcile first.

Apply uses the same `POST /api/stack-apply`. Response `{ok:true, mode:"update", job, state, name}` (job action `stack-update`). Atlas records `stack_actions` as `update` / `queued`. Job result: `{name, changed, changes:[{file,status}], backup, ssh_port, ssh_users[], steps[]}`. `changed: false` is a **neutral info** (“No changes”), not an error. Failure: `{step, output, done[], rollback[], backup}` or the string `drift: …`.

---

## Reconcile

When `drift` is true, Edit is disabled and **Reconcile** opens a dialog with two one-line options:

- Adopt live files (only works if they still match a blueprint)
- Re-render from the stored spec (overwrites the live files; a backup is kept)

`POST /api/stacks/{name}/reconcile` with `{mode: "adopt-live" | "re-render"}`.

- `adopt-live` is synchronous: `{name, mode, spec, ssh_port}`. Atlas records `reconcile` / `ok`. 409 if live files do not match a blueprint render.
- `re-render` is a job: `{ok, job, state, name, mode}` (action `stack-reconcile`). Atlas records `reconcile` / `queued`. Job result is like update plus `mode` and `dropped` (people removed from the central list). Re-render on a non-sandbox stack needs a type-the-name confirm.

`404` `stack not found`. `409` `stack busy`.

---

## People (`/stacks/people`)

Central SSH people (`[{name, uid}]`, no keys) and the stacks that are out of sync (`out_of_sync: true`), each with `dropped[]` and `errors[]`. **Sync** per stack and **Sync all** (`POST /api/stack-people-sync`, `{name?}`; omit `name` for every out-of-sync stack). Response `{"jobs":[{name, job, state} | {name, error}]}`. Atlas records `people-sync` / `queued` (or `error`) per row. Job result is like update plus `dropped`. The result panel lists dropped people.

---

## Trash (`/stacks/trash`)

Card grid of trash entries, grouped by type: name, type, deleted at, purge after (days left), volumes, dump path only if present. **Restore** confirms, then `POST /api/stack-trash/{name}/restore`. Atlas records `stack_actions` as `trash-restore` / `queued` with the job id. On success: toast + link to the restored stack. On failure: failed `step` + output. `404` `not in trash`; `409` stack busy (detail may be `trash folder missing`, `<path> already exists`, `volume <v> missing`, `ssh port <n> is in use`).

---

## Atlas API

Admin only. 404 when the flag is off. Client: `app/Domain/Stacks/StackApiClient.php`. Timeouts: restart 65 s (15 s when async), logs 30 s, plan 30 s, apply 30 s, delete 30 s, trash list / restore 30 s, reconcile 30 s, people-sync 30 s, people / fix-permissions 15 s. Every Stella body includes a random `nonce`. Empty `php_ini` is `{}`, never `[]`. Delete always sends `final_dump` as JSON `false`.

| Atlas | Stella action |
|-------|---------------|
| `GET /api/stacks` | `stack-list` |
| `GET /api/stacks/{name}` | `stack-get` (+ local history) |
| `GET /api/stacks/{name}/logs` | `stack-logs` |
| `POST /api/stacks/{name}/restart` | `stack-restart` |
| `POST /api/stacks/{name}/fix-permissions` | `stack-fix-permissions` |
| `POST /api/stacks/{name}/delete` | `stack-delete` (`{confirm, final_dump: false}`; confirm must equal name) |
| `POST /api/stacks/{name}/reconcile` | `stack-reconcile` (`{mode: adopt-live\|re-render}`) |
| `GET /api/stack-jobs` / `{id}` | `job-list` / `job-get` |
| `GET /api/stack-people` | `people-list` |
| `POST /api/stack-people-sync` | `stack-people-sync` (`{name?}`; omit `name` for all out-of-sync) |
| `POST /api/stack-plan` | `stack-plan` |
| `POST /api/stack-apply` | `stack-apply` (create or update; records `apply` or `update`) |
| `GET /api/stack-trash` | `stack-trash-list` |
| `POST /api/stack-trash/{name}/restore` | `stack-trash-restore` |

Use `/api/stack-jobs`, `/api/stack-plan`, `/api/stack-people`, `/api/stack-people-sync`, `/api/stack-apply`, `/api/stack-trash` — not `/api/stacks/jobs` and so on (`jobs` / `people` / `trash` would match `{name}`).

Name `^[a-z0-9][a-z0-9.-]{0,252}$`, service `^[a-z0-9][a-z0-9_-]{0,62}$`, job id `^[0-9a-f]{32}$` — validated in Atlas before the call. Plan names are not pre-validated so Stella can return sandbox / reserved-name errors.

Mapped errors: 401/403 → `stack-api rejected the request`; 400/404 detail passed through; 409 → stack busy; 502 → docker error (results passed through); connection → `stack-api unreachable`. A handled plan or apply rejection is HTTP 200 with `ok` + `errors`.

Never log the token, signature, key contents, log lines, spec keys, or response files.

---

## Not in this UI

Stella still discovers Laravel and .NET stacks for list/get/restart/logs. Create has no blueprint for them yet. PHP version change, labels, basic auth, staging guard, and cron are still open. Roadmap: [`../stella-server/stack-api.md#roadmap`](../stella-server/stack-api.md#roadmap).
