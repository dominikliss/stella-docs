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
| Plan a new WordPress stack (dry-run compose / files / Caddy; mode `create`) | API only — the create drawer no longer plans first |
| Create a WordPress stack (queued apply job; in-grid card with the current step only) | yes |
| WordPress login after create (URL, user, email, password once from the job result, table prefix) | yes |
| Edit a managed WordPress stack (apply; job action `stack-update`) | yes |
| Reconcile drift (`adopt-live` sync or `re-render` job) | yes |
| People list + out-of-sync stacks; sync one or all | yes |
| Recent stack-api jobs + Atlas `stack_actions` history | yes |
| Delete a managed WordPress stack (queued job; trash for `retention_days`) | yes |
| Trash list + restore (queued job) | yes |
| Trash purge now (queued job; frees the name) | yes |
| Restore from an Atlas backup | **not built** (`create` + `restore` is the planned replacement for wp-staging-api) |
| Create Laravel or .NET stacks | **not built** (list/get still show those types) |
| PHP version change | **not built** |

List polls every 30 s and always refetches when `/stacks` mounts (so a restore from Trash is visible without a full reload). The grid stays empty until the real cards arrive — no skeleton placeholders. Detail polls every 30 s; jobs every 10 s; a watched job every 2 s until `ok` / `error`.

---

## List (`/stacks`)

Card grid grouped by WordPress → Laravel → .NET → anything else, then name. Each card has a scaled live preview when the stack name is a hostname (`https://{name}` in a sandboxed iframe), a status dot, the name, a **drift** badge when `drift` is true, and a restart control. While the iframe is loading, and during create or delete, the preview pane stays dark — no type globe. Create and delete overlays, including the scan line, are clipped to that pane (not the window chrome or the name row). Clicking a card opens the stack in a sidebar (URL `/stacks/$name`) instead of a separate page. Container chips and sandbox/managed pills stay off the card. Status is derived from container `state` (dot: all running / degraded / down). Open bind IPs (`0.0.0.0` / `::`) are highlighted in the sidebar, not the list.

**Stacks / People / Trash** tabs. **New stack** opens the create drawer. Create is WordPress-only in the UI.

---

## Create

The form submits **Create** (`POST /api/stack-apply`) with no dry-run step. Create is enabled when the name is valid and at least one SSH user is selected. The drawer closes as soon as Stella queues the job; the WordPress grid shows a placeholder card in the sorted slot the new stack will occupy (by name, among existing WordPress cards). That card shows only the current apply step (files → compose-up → Caddy → HTTP wait → WordPress install → permissions → state), not the full checklist, with no live iframe under the overlay. The scan animation runs only inside the preview pane. Stella still accepts `POST /api/stack-plan`; Atlas create and edit no longer call it. Changing name, PHP, SSH users, or ini rows just changes what Create sends.

| Field | Rule |
|-------|------|
| Name | One DNS label; Atlas appends `.staging.foxcraft.digital`. Pattern `^[a-z0-9]([a-z0-9-]{0,61}[a-z0-9])?$`. Reserved: `atlas`, `api`, `stella`, `edison`, `york`, `www`, `mail`, `stella-*` |
| Type | WordPress (locked) |
| PHP | `8.4` or `8.5` (default `8.5`). Locked on edit — Stella changes PHP with the PHP version job, not with update |
| SSH users | At least one name from `GET /api/stack-people`. Defaults to every returned person. uid + key come from Stella `people/`, not the form |
| PHP settings | Always shown with blueprint defaults and a value dropdown per key. Overlay from `spec.php_ini` on edit. Keys: `memory_limit` `256M`, `upload_max_filesize` `64M`, `post_max_size` `64M`, `max_execution_time` `300`, `max_input_time` `60`, `max_input_vars` `1000`, `display_errors` `Off`, `date.timezone` `Europe/Vienna`, `session.gc_maxlifetime` `1440`, `opcache.enable` `1`, `opcache.memory_consumption` `128`, `opcache.max_accelerated_files` `10000`, `opcache.validate_timestamps` `0`, `opcache.revalidate_freq` `2` |

Plan response (always HTTP 200 when handled) includes `mode: "create"`: path, SSH port (or “no SSH”), SSH users, notes, and artifacts (`docker-compose.yml`, extra files, Caddy site block). `ok: false` with a rendered compose means policy failed (e.g. a disallowed `php_ini` key). An existing **managed** stack is an update (below), not a create. Unmanaged existing stacks / folders / Caddy files are still refused. A name in the trash is rejected (`name: <name> is in the trash until <ts>; restore it, purge it, or wait for the purge`). Atlas shows a link to `/stacks/trash` so the name can be restored or purged immediately. A sandbox principal may plan/apply any valid `<label>.staging.foxcraft.digital` name; after apply it owns that stack and it shows up in its list.

Create sends `POST /api/stack-apply` (no confirm dialog). The drawer closes immediately and a placeholder card appears in the list; Atlas records `stack_actions` as `apply` / `queued` once Stella returns a job id (`mode: "create"`). The card polls the job every 2 s and shows only the current apply step; when the job reports `done` / `step` those win over the time estimate. URL, SSH, and WordPress login stay off the placeholder. The placeholder stays in that slot until the job is `ok` (even if Stella already lists the stack after compose-up). Then it is replaced by the normal card and the live preview iframe is remounted so it does not keep the installer / database-error page from mid-install. A toast reports success or failure. Immediate apply rejections (trash hold, policy) are also toasts. The stack sidebar still shows a **WordPress login** block from the latest `stack-apply` job when `result.admin` is present: login URL `{result.url}/wp-admin/`, user, email, table prefix, and the admin password (masked, reveal + copy). If `admin.password` is `null`, the password row is hidden and the block says “already installed”. The password is shown only from the stack-api job result; Atlas does not write it to `stack_actions`, history, or any Atlas table, and does not log it. If the create job is `ok` but `result.admin` is missing, Atlas warns that WordPress was not installed (the public site is still the installer). Apply is a heavy job (one at a time on Stella).

New sites are IP-restricted in Caddy (`STACK_API_SITE_ALLOW_IPS`: Proton, york, Atlas prod). Wildcard cert for `*.staging.foxcraft.digital` — the site block has no `tls` section.

---

## Detail (sidebar on `/stacks/$name`)

One 640px drawer with three tabs. Create still uses its own drawer.

| Tab | Contents |
|-----|----------|
| Stack | Sandbox/managed/drift tags, WordPress login, edit form (when `managed` and `spec` are present), danger zone |
| Containers | Two-column card grid (image, state, status, started, restarts, health, bind ports, logs, restart) |
| Jobs | Watched job, last restart, recent stack-api jobs, Atlas history |

| Action | Notes |
|--------|-------|
| Logs | Per service, from a container card. Docker stdout/stderr, timestamps on. 50 / 200 / 1000 lines (API 1–1000, default 200). Panel scrolls to the newest line. Truncated containers are marked. Log bodies are never written to the Stella audit log |
| Restart service | Confirm dialog. Always queued (`async`); Atlas watches the job |
| Restart stack | On the card and in the sidebar header. Always queued. Type-the-name confirm unless the stack is sandbox. Stella restarts `db` first |
| Edit | On the Stack tab, not a second drawer. Form is prefilled from `spec`. Locked when `drift` is true |
| Reconcile | Shown on the Stack tab when `drift` is true. Dialog: adopt live files, or re-render from the stored spec. Re-render on a non-sandbox stack needs a type-the-name confirm |
| Delete | Shown only when `managed` is true (Stack tab danger zone). Type-the-name confirm. Confirm starts the job immediately (card Deleting overlay in the preview pane, no globe) while the confirm dialog and sidebar animate out. Stack goes to the trash for `retention_days` and can be restored. Queued job; toast on settle. On failure: toast with `detail`. Atlas always sends `final_dump: false` (no SQL copy; restore uses volumes) |

`drift` is `true` / `false` / `null` (`null` = not managed). A **drift** badge is on the card and in the sidebar when it is true.

Busy stack (sync restart or any job already running) → `409` “Stack is busy — another action is running.” Delete also maps 409 `stack is not managed by stack-api` / `stack is already in the trash` / `stack busy` (message stays `stack busy`; detail is passed through). Adopt-live 409 `live files do not match a blueprint render; use re-render` is shown in the reconcile dialog (message still `stack busy`; detail is passed through).

**Recent jobs** come from stack-api (`limit` 10 on the page). **History** is Atlas `stack_actions` (last 50): restart, apply, update, reconcile, people-sync, fix-permissions, delete, trash-restore, trash-purge. Queued rows store the job id in `detail` and settle to `ok` / `error` when the job is fetched (detail page, job list, or job get). `plan`, `people`, and trash list do not write `stack_actions`. Stella keeps its own append-only audit log.

---

## Edit (Apply)

Managed stacks only. The Stack tab embeds the same form as create, prefilled from `GET /api/stacks/{name}` `spec`. **Name** and **type** are locked. **PHP** is locked with the hint “change via PHP version (coming)” — Stella rejects a PHP change on update (`php: is 8.5; the PHP version is changed with the PHP version job, not with update`). There is no Plan step. **Apply** is enabled when the form is valid (at least one SSH user) and the stack is not drifted. Confirm, then `POST /api/stack-apply`.

Stella still accepts `POST /api/stack-plan` for a dry-run (`mode: "update"` plus `drift`, `changes`, notes, artifacts). Atlas does not call it. `ok: false` with `drift: live files differ from the last apply; reconcile first` means Apply is the wrong action — reconcile first. The form is locked until then.

Apply response `{ok:true, mode:"update", job, state, name}` (job action `stack-update`). Atlas records `stack_actions` as `update` / `queued`. Job result: `{name, changed, changes:[{file,status}], backup, ssh_port, ssh_users[], steps[]}`. `changed: false` is a **neutral info** (“No changes”), not an error. Failure: `{step, output, done[], rollback[], backup}` or the string `drift: …`.

---

## Reconcile

When `drift` is true, the edit form is locked and **Reconcile** opens a dialog with two one-line options:

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

Card grid of trash entries, grouped by type: name, type, deleted at, purge after (days left), volumes, dump path only if present. **Restore** confirms, then `POST /api/stack-trash/{name}/restore`. Atlas records `stack_actions` as `trash-restore` / `queued` with the job id. On success: toast, and the stacks list is refetched until the restored name appears (so switching back to Stacks does not wait for a reload). On failure: failed `step` + output. `404` `not in trash`; `409` stack busy (detail may be `trash folder missing`, `<path> already exists`, `volume <v> missing`, `ssh port <n> is in use`).

**Purge now** type-the-name confirms, then `POST /api/stack-trash/{name}/purge` (`{confirm}` must equal name). Backed by stack-api `stack-trash-purge` (live 2026-11-14). Atlas records `stack_actions` as `trash-purge` / `queued`. Destroys volumes and the trash folder immediately so the name can be created again. On success: toast + “the name is free”. `400` `confirm must equal name`; `404` `not in trash`; `409` stack busy.

---

## Atlas API

Admin only. 404 when the flag is off. Client: `app/Domain/Stacks/StackApiClient.php`. Timeouts: restart 65 s (15 s when async), logs 30 s, plan 30 s, apply 30 s, delete 30 s, trash list / restore / purge 30 s, reconcile 30 s, people-sync 30 s, people / fix-permissions 15 s. Every Stella body includes a random `nonce`. Empty `php_ini` is `{}`, never `[]`. Delete always sends `final_dump` as JSON `false`.

| Atlas | Stella action |
|-------|---------------|
| `GET /api/stacks` | `stack-list` (`Cache-Control: no-store`) |
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
| `POST /api/stack-trash/{name}/purge` | `stack-trash-purge` (`{confirm}` must equal name) |

Use `/api/stack-jobs`, `/api/stack-plan`, `/api/stack-people`, `/api/stack-people-sync`, `/api/stack-apply`, `/api/stack-trash` — not `/api/stacks/jobs` and so on (`jobs` / `people` / `trash` would match `{name}`).

Name `^[a-z0-9][a-z0-9.-]{0,252}$`, service `^[a-z0-9][a-z0-9_-]{0,62}$`, job id `^[0-9a-f]{32}$` — validated in Atlas before the call. Plan names are not pre-validated so Stella can return sandbox / reserved-name errors.

Mapped errors: 401/403 → `stack-api rejected the request`; 400/404 detail passed through; 409 → stack busy; 502 → docker error (results passed through); connection → `stack-api unreachable`. A handled plan or apply rejection is HTTP 200 with `ok` + `errors`.

Never log the token, signature, key contents, log lines, spec keys, or response files.

---

## Not in this UI

Stella still discovers Laravel and .NET stacks for list/get/restart/logs. Create has no blueprint for them yet. PHP version change, labels, basic auth, staging guard, and cron are still open. Roadmap: [`../stella-server/stack-api.md#roadmap`](../stella-server/stack-api.md#roadmap).
