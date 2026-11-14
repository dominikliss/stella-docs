# stack-api (container management for Atlas)

**Status:** live since 2026-11-04. As of 2026-11-14 (`main.py`): signed `list` / `get` / `restart` (sync + async) / `logs` / `plan` / `apply` (create + update of `php_ini` / `ssh_users`) / `reconcile` / `fix-permissions` / `delete` / trash list / trash restore / trash purge-now / people list / people sync, `stack_state` + drift detection + job table with per-stack locks, append-only audit log, hourly trash purge. WordPress auto-install on create (2026-11-14). Atlas UI on atlas.dev: [`../atlas/stacks.md`](../atlas/stacks.md) (list/search, logs, restart, WordPress Plan → Create / Edit, drift reconcile, people sync, fix-permissions, delete + trash/restore/purge, jobs, history). First stack created end to end from Atlas on 2026-11-08: `sandbox.staging.foxcraft.digital` (adopted into `stack_state` 2026-11-14, owner `atlas-dev`). Not yet: PHP version change, restore from an Atlas backup, Laravel/.NET blueprints.

**URL:** `https://stella-stack-api.foxcraft.digital`

**Purpose:** the single entry point through which Atlas creates, changes, restarts and deletes app stacks (WordPress, Laravel, .NET) on Stella. Atlas holds the spec; stack-api renders compose files from its own blueprints and applies them. Staging from an Atlas backup becomes a `create` + `restore` job here; [`wp-staging-api.md`](wp-staging-api.md) is superseded and will not be deployed as a separate service.

Infra: [`infrastructure.md`](infrastructure.md). System map: [`../integration/system-overview.md`](../integration/system-overview.md).

**Rule (decided 2026-11-04):** stacks under `/opt/apps/` are not edited by hand. Changes go through Atlas → stack-api. Blueprint changes go through the templates in the repo.

---

## Files on Stella

```
/opt/services/stack-api/
  ├── Dockerfile          ← multi-stage: docker CLI from docker:cli, compose plugin from docker/compose-bin:latest (→ /usr/libexec/docker/cli-plugins/docker-compose), python:3.13-slim + openssh-client + acl, ENV DOCKER_CONFIG=/tmp/docker (container is read-only), COPY people/ (overlaid by the bind mount) and COPY app/. Compose in the container: v5.7.0
  ├── requirements.txt    ← fastapi, uvicorn, httpx, pyyaml
  ├── .dockerignore       ← .env, allowed_signers, data, images
  ├── .env                ← chmod 600; see keys below
  ├── allowed_signers     ← mounted read-only, not baked into the image; atlas-prod + atlas-dev, both namespaces="stack"
  ├── data/               ← host dir; SQLite at /data/stack-api.sqlite in the container (code default; STACK_API_DB is not in .env)
  ├── people/
  │     ├── users.conf    ← central SSH people list (public keys only); bind-mounted read-only
  │     └── keys/<name>.pub
  ├── app/main.py         ← API, auth, jobs, apply, update, reconcile, delete/trash, people sync, rollback, permissions
  ├── app/stackspec.py    ← spec validation, people loader, WordPress blueprint, Caddy block, policy, YAML dump
  └── images/ssh-php/     ← shared SSH image (not in the API container)
        ├── Dockerfile
        ├── atlas-sync-users
        └── entrypoint.sh
```

Never write the token value into docs. Backups on Stella: `app/main.py.bak-20261108`, `-20261108b`, `-20261108c`, `-20261108d`; `app/main.py.bak-20261114` (before delete/trash), `app/main.py.bak-20261114b` (before update), `app/main.py.bak-20261114c` (before purge-now). The live `app/main.py` is the version with WordPress install (`install_wordpress` / `wp_cli`, random table prefix) plus purge-now. `app/stackspec.py.bak-20261108`, `-20261108b`, `-20261108c`; stack-api `Dockerfile.bak-20261108`. stack-api `Dockerfile` and `requirements.txt` are unchanged since 2026-11-08. Shared SSH image: `images/ssh-php/Dockerfile.bak-20261114` (before mysqli).

`.env` keys (no values): `STACK_API_TOKEN`, `STACK_API_ALLOWED_IPS`, `STACK_API_SANDBOX_STACKS`, `STACK_API_SITE_ALLOW_IPS`, `STACK_API_DOMAINS`, `STACK_API_TRASH_DAYS`, `STACK_API_PHP_VERSIONS`. `STACK_API_DB` is **not** in `.env` — the code default `/data/stack-api.sqlite` is used. Optional with defaults: `STACK_API_PHP_VERSIONS` (`8.5` if unset; live `.env` is `8.4,8.5`), `STACK_API_TRASH_DAYS` (`7`), `STACK_API_HEAVY_JOBS` (`1`), `STACK_API_CADDY_CONTAINER` (`services-caddy-1`), `STACK_API_CADDY_SITES` (`/caddy-sites`), `STACK_API_PEOPLE` (`/app/people`), `STACK_API_WP_ADMIN_USER` (`foxcraft.digital`), `STACK_API_WP_ADMIN_EMAIL` (`projects@foxcraft.digital`).

`people/users.conf` + `people/keys/<name>.pub` are bind-mounted (`./stack-api/people:/app/people:ro`). Current: `dominik` 1000, `pawel` 1001, `edison` 1002 (copied from hypershape 2026-11-08). Changing a key = edit `people/` — no stack-api image rebuild. Running stacks catch up via `people/sync` (or reconcile re-render).

Values: `STACK_API_SANDBOX_STACKS` = `dominikliss.foxcraft.digital,sandbox.staging.foxcraft.digital`. `STACK_API_SITE_ALLOW_IPS` = `194.126.177.181` (Proton), `78.47.218.190` (york), `2.31.42.13` (Atlas prod), decided 2026-11-08. `STACK_API_DOMAINS` = `staging.foxcraft.digital`. `STACK_API_TRASH_DAYS` = `7`. `STACK_API_PHP_VERSIONS` = `8.4,8.5`.

---

## Compose service (`/opt/services/docker-compose.yml`)

```yaml
  stack-api:
    build: ./stack-api
    container_name: stack-api
    env_file:
      - ./stack-api/.env
    volumes:
      - ./stack-api/allowed_signers:/app/allowed_signers:ro
      - ./stack-api/people:/app/people:ro
      - ./stack-api/data:/data
      - /var/run/docker.sock:/var/run/docker.sock
      - /opt/apps:/opt/apps
      - ./caddy/sites:/caddy-sites
    read_only: true
    tmpfs:
      - /tmp
    networks:
      - stack
    restart: unless-stopped
```

`/opt/apps` is the same path inside and outside (was `/opt/apps/wordpress` until 2026-11-14) — required: compose resolves `./` paths in the container, the daemon mounts them on the host, the working_dir label must match stack discovery, and delete moves the folder to `/opt/apps/.trash/<name>-<ts>`.

Top-level network `stack` (Docker name `services_stack`). **Only `stack-api` and `services-caddy-1` are on it.** stack-api is deliberately **not** on `edge`: any container on `edge` could otherwise reach it directly and fake `X-Forwarded-For`. `caddy` has `default`, `edge`, and `stack`.

No published ports.

With the socket mounted, stack-api is **root-equivalent** on Stella.

---

## Three protection layers

| Layer | Where | Detail |
|---|---|---|
| IP allowlist | Caddy **and** app | Caddy: `@blocked not remote_ip 2.31.42.13 95.217.144.93 172.20.0.1` → 403. App: rightmost `X-Forwarded-For` must be in `STACK_API_ALLOWED_IPS` (same three IPs); missing header → 403. Applies to every path, including `/health` |
| Secret token | app | Header `X-Stack-Token`, constant-time compare against `STACK_API_TOKEN` (64 hex chars, `openssl rand -hex 32`, min. 32 chars enforced at startup) |
| SSH signature | app | See below |

Every failed check answers a generic `401 {"detail":"unauthorized"}` or `403 {"detail":"forbidden"}`. The reason is only logged (`denied reason=…` / `denied ip=…`).

`2.31.42.13` = Atlas prod. `95.217.144.93` = Stella host. `172.20.0.1` = Docker `edge` gateway. Verified 2026-11-04: a container on `edge` (atlas.dev app) calling the public URL is routed through Docker's proxy; Caddy sees `172.20.0.1` (`netstat` inside `services-caddy-1`: peer `::ffff:172.20.0.1`), not `95.217.144.93`. From the Stella host itself the source stays `95.217.144.93`. `172.20.0.1` means any container on `edge` — same trust level as the Stella IP; token + signature + sandbox scope remain the real gates. **Gotcha:** test container-to-API paths from inside the container, not only from the host.

### Signed requests

```
payload   = "<action>:<unix_ts>:<sha256(request)>"
POST body = {"payload": …, "signature": …, "request": "<JSON string>"}
```

- `ssh-keygen -Y find-principals` (signer → principal), then `ssh-keygen -Y verify -n stack -I <principal>`
- **Namespace `stack`** — not `deploy`. Signatures for deploy-api / health-api are never valid here.
- Unknown principals are rejected.
- 60 s window, replay cache on the signature hash (in memory, entries kept 120 s)
- The body hash binds the request JSON to the signature
- `request` must be a JSON object
- **Every request must include a random `nonce`.** Ed25519 signatures are deterministic: two identical `{action, timestamp, request}` in the same second produce the same signature and are rejected as replay (seen 2026-11-04 01:29, audit row 19). Atlas `StackApiClient` adds `nonce` (`bin2hex(random_bytes(16))`) to every body before encoding.

### Principals and sandbox

| Principal | Key | Scope |
|---|---|---|
| `atlas-prod` | host `atlas`, `/home/atlas/.ssh/atlas-stack-api` (owner `atlas`, no passphrase, comment `atlas-stack-api`) | `all` |
| `atlas-dev` | atlas.dev `/var/www/.ssh/id_ed25519` (same file as the atlas-dev deploy key) | `sandbox` |

`PRINCIPAL_SCOPES = {"atlas-prod": "all", "atlas-dev": "sandbox"}`. `allowed_signers` has both lines, both `namespaces="stack"`. The shared Atlas prod deploy key (`/home/atlas/.ssh/id_ed25519`, principal `ddashboard`) is **not** trusted by stack-api.

`STACK_API_SANDBOX_STACKS` in stack-api `.env` is the sandbox allowlist — currently `dominikliss.foxcraft.digital,sandbox.staging.foxcraft.digital` (the stack name is the folder name, not `dominikliss`). A sandbox principal's **visible** set is `STACK_API_SANDBOX_STACKS` ∪ stacks it **owns** (`stack_state.created_by` = that principal). Any other name is `404`. A sandbox principal may **plan** / **apply** any valid `<label>.staging.foxcraft.digital` name; after a successful apply it owns that stack and it shows up in its list. `list` / `get` include `"sandbox"` and `"managed"` on each stack; `ping` includes `"scope"`.

**Decision (2026-11-04):** `dominikliss.foxcraft.digital` is on the sandbox allowlist so `atlas-dev` can see it. It is **not** managed (hand-built; backfill did not adopt it), so it cannot be deleted through stack-api. **2026-11-08:** first stack created end to end from Atlas: `sandbox.staging.foxcraft.digital`. New stacks are `<label>.staging.foxcraft.digital`. **2026-11-14:** that stack was adopted into `stack_state` (owner `atlas-dev`).

**Incident 00:51 (2026-11-04):** the `atlas-dev` signer was added to `allowed_signers` before the sandbox-restricting code was in the running container. atlas.dev then restarted the whole `osgar` stack (audit row 11). No damage. **Rule:** never add a principal before the restricting code is verified in the running container.

### Atlas `.env`

```
STELLA_STACK_API_URL=https://stella-stack-api.foxcraft.digital
STELLA_STACK_API_TOKEN=<value>
STELLA_STACK_API_ENABLED=false
STELLA_STACK_API_KEY_PATH=
```

Atlas client (2026-11-04, logs/jobs/nonce/plan/apply/people/fix-permissions 2026-11-08, delete/trash 2026-11-14, update/drift/reconcile/people-sync 2026-11-14): `app/Domain/Stacks/StackApiClient.php`. Flag default **false** — `/api/stacks` is 404 and the nav item is hidden. Setting `stack_api_ssh_key_path` (Settings → GitHub, next to the deploy key path) overrides `STELLA_STACK_API_KEY_PATH`.

| Instance | Enable | Key path |
|---|---|---|
| atlas.dev | **on** 2026-11-04 (`STELLA_STACK_API_ENABLED=true` in `src/.env`) | `/var/www/.ssh/id_ed25519` |
| Atlas prod | **off** (flag stays false) | `/home/atlas/.ssh/atlas-stack-api` when enabled |

Admin role only. Atlas records restarts, apply, update, reconcile, people-sync, fix-permissions, delete, trash-restore, and trash-purge in `stack_actions` (`user_id`, stack, service, result, detail, created_at`). Queued rows store the job id in `detail` and settle to `ok` / `error` when the job is fetched. stack-api keeps its own audit log.

### Atlas client

`StellaSshSigner::sign($payload, 'stack', $keyPath)` — namespace **`stack`**, not `deploy`. Payload `<action>:<unix_ts>:<sha256(request_json)>`. POST `{payload, signature, request}` where `request` is the exact JSON string that was hashed (encoded once; integers stay integers, booleans stay booleans). Header `X-Stack-Token`. Restart timeout 65 s (15 s when `async`). Logs timeout 30 s. Plan timeout 30 s. Apply timeout 30 s. Delete timeout 30 s. Trash list / restore / purge timeout 30 s. Reconcile timeout 30 s. People-sync timeout 30 s. People list and fix-permissions 15 s. A random `nonce` is sent on every request. Empty `spec` / `php_ini` are JSON objects (`{}`), never `[]`. Delete always sends `final_dump` as JSON `false`.

| Atlas | stack-api |
|---|---|
| `GET /api/stacks` | `POST /stacks/list` action `stack-list` request `{nonce}` |
| `GET /api/stacks/{name}` | `POST /stacks/get` action `stack-get` request `{"name", nonce}` |
| `GET /api/stacks/{name}/logs` | `POST /stacks/logs` action `stack-logs` request `{"name","service","lines"?, nonce}` (`lines` 1–1000, default 200) |
| `POST /api/stacks/{name}/restart` | `POST /stacks/restart` action `stack-restart` request `{"name", "service"?, "async"?, nonce}` — `async` only when true |
| `POST /api/stacks/{name}/fix-permissions` | `POST /stacks/fix-permissions` action `stack-fix-permissions` request `{"name", nonce}` |
| `POST /api/stacks/{name}/delete` | `POST /stacks/delete` action `stack-delete` request `{"name","confirm","final_dump", nonce}` (`confirm` must equal `name`; Atlas always sends `final_dump` as JSON `false`) |
| `POST /api/stacks/{name}/reconcile` | `POST /stacks/reconcile` action `stack-reconcile` request `{"name","mode", nonce}` (`mode` is `adopt-live` or `re-render`) |
| `GET /api/stack-jobs` | `POST /jobs/list` action `job-list` request `{"name"?, "limit"?, nonce}` (`limit` 1–100, default 20) |
| `GET /api/stack-jobs/{id}` | `POST /jobs/get` action `job-get` request `{"id", nonce}` |
| `GET /api/stack-people` | `POST /people/list` action `people-list` request `{nonce}` |
| `POST /api/stack-people-sync` | `POST /people/sync` action `stack-people-sync` request `{"name"?, nonce}` (omit `name` for every out-of-sync stack) |
| `POST /api/stack-plan` | `POST /stacks/plan` action `stack-plan` request `{"spec", nonce}` (dry-run; no `stack_actions` row; create or update) |
| `POST /api/stack-apply` | `POST /stacks/apply` action `stack-apply` request `{"spec", nonce}` (create or update) |
| `GET /api/stack-trash` | `POST /stacks/trash/list` action `stack-trash-list` request `{nonce}` |
| `POST /api/stack-trash/{name}/restore` | `POST /stacks/trash/restore` action `stack-trash-restore` request `{"name", nonce}` |
| `POST /api/stack-trash/{name}/purge` | `POST /stacks/trash/purge` action `stack-trash-purge` request `{"name","confirm", nonce}` (`confirm` must equal `name`) |

Use `/api/stack-jobs`, not `/api/stacks/jobs` — `jobs` would match `{name}`. Use `/api/stack-plan` / `/api/stack-people` / `/api/stack-people-sync` / `/api/stack-apply` / `/api/stack-trash`, not `/api/stacks/plan` and so on (`jobs` / `people` / `trash` would match `{name}`). Name `^[a-z0-9][a-z0-9.-]{0,252}$`, service `^[a-z0-9][a-z0-9_-]{0,62}$`, job id `^[0-9a-f]{32}$`, lines 1–1000, limit 1–100 — validated in Atlas before the call. Plan names are not pre-validated so stack-api can return sandbox / reserved-name errors. Mapped errors: 401/403 → `stack-api rejected the request`; 404 detail passed through; 400 detail passed through (`invalid input` when there is no detail); 409 → `stack busy`; 502 → `docker error` (results passed through); connection → `stack-api unreachable`. A handled plan or apply rejection is always HTTP 200 (`ok` + `errors`; `ok` can be false with a rendered compose). Never log the token, signature, key contents, log lines, spec keys, or response files. Async restart, apply, update, re-render reconcile, people-sync, fix-permissions, delete, trash-restore, and trash-purge record `stack_actions` with result `queued` and the job id in `detail` (`service` is the action for apply / update / reconcile / people-sync / fix-permissions / delete / trash-restore / trash-purge). `adopt-live` records `reconcile` / `ok`. `plan`, `people`, and trash list do not write `stack_actions`. Atlas `ssh_users` is a required list of person names (min 1).

---

## Stack discovery

A container belongs to a stack if its label `com.docker.compose.project.working_dir` is exactly `/opt/apps/<type>/<name>` with `type` in `{wordpress, laravel, dotnet}` and `name` matching `^[a-z0-9][a-z0-9.-]{0,252}$`. Stack name = folder name, type = parent folder. Everything in `/opt/services` (Caddy, deploy-api, health-api, …) is invisible and cannot be restarted through stack-api.

## Endpoints (current)

Signed endpoints: action = payload prefix. Auth is IP + token + signature unless noted.

| Path | Action | Request | Response |
|---|---|---|---|
| `GET /health` | — | — (IP only) | `{"status":"ok"}` |
| `POST /ping` | `stack-ping` | JSON object + `nonce` | `{"ok":true,"principal":…,"scope":…,"request":{…}}` |
| `POST /stacks/list` | `stack-list` | `{nonce}` | `{"stacks":[{name, type, path, sandbox, managed, drift (true\|false\|null), containers:[{id (12 chars), name, service, image, state, status, ports:[{ip, public, private, type}]}]}]}`. `drift` is `null` when the stack is not managed |
| `POST /stacks/get` | `stack-get` | `{"name", nonce}` | one stack including `sandbox`, `managed`, `drift`, `spec` (`{type, name, php, ssh_users[], php_ini{}}` or null), `ssh_port`; containers also have `started_at`, `restart_count`, `health`. `400` `invalid name`, `404` `stack not found` (also for a sandbox principal asking for a stack that is not visible) |
| `POST /stacks/restart` | `stack-restart` | `{"name", "service"?, "async"?, nonce}` — `async` only when true | Sync (unchanged without `async`): restarts one service or the whole stack (`db` first); Docker restart `t=10`. `{"name","results":[{service, name, ok}]}`. `async: true` → `{job, state:"queued"}`. `404` `service not found`; `409` `stack busy`; `502` with `results` if any restart failed |
| `POST /stacks/logs` | `stack-logs` | `{name, service, lines?, nonce}` (`lines` 1–1000, default 200) | `{name, service, containers:[{name, state, truncated, lines:[{ts, stream, text}]}]}`. Container stdout/stderr via the Docker logs API (demuxed, timestamps on), max 2 MB per container, each line cut to 4000 chars. Log content is never written to the audit log (detail is only `lines=N`). `404` `service not found` |
| `POST /jobs/get` | `job-get` | `{id, nonce}` (32 lowercase hex) | `{id, action, stack, service, principal, state (queued / running / ok / error), created_at, started_at, finished_at, result, detail}`. `stack-apply` success `result` also has `table_prefix` and `admin: {user, email, password}` (`password` is `null` if WordPress was already installed; stored in `jobs.result`, not in audit/`detail`). `404` `job not found`, also when the job's stack is not visible to the principal. **Exception:** a sandbox principal can still read jobs of stacks it created after they were purged (needed to poll the purge job) |
| `POST /jobs/list` | `job-list` | `{name?, limit?, nonce}` (`limit` 1–100, default 20; omit `name` when null) | `{jobs:[job, …]}` newest first, filtered by principal scope |
| `POST /people/list` | `people-list` | `{nonce}` | `{people:[{name, uid}], stacks:[{name, out_of_sync (bool\|null), dropped[], errors[]}]}` — no keys |
| `POST /people/sync` | `stack-people-sync` | `{name?, nonce}` (omit `name` for every out-of-sync stack) | `{"jobs":[{name, job, state} \| {name, error}]}`. Heavy job per stack. Job result like update plus `dropped` |
| `POST /stacks/plan` | `stack-plan` | `{spec, nonce}` | Always 200 for a handled plan. Writes nothing. Rejected: `{ok:false, errors:[string]}`. Create: `{ok, errors, mode:"create", name, type, path, ssh_port (null without users), ssh_users, compose (YAML, multi-line strings as `\|` blocks), files, caddy, notes}`. Create notes: DB passwords and a random table prefix are generated on apply; WordPress is installed on apply with title = first label of the name, admin `foxcraft.digital` `<projects@foxcraft.digital>`; the admin password is returned once in the job result. Update (existing **managed** stack): same plus `drift` and `changes:[{file, status: added\|changed\|removed, diff}]`. Update errors include `php: is 8.5; the PHP version is changed with the PHP version job, not with update` and `drift: live files differ from the last apply; reconcile first`. `ok` can be false with a rendered result when policy fails |
| `POST /stacks/apply` | `stack-apply` | `{spec, nonce}` | Rejected: `{ok:false, errors:[string]}` (HTTP 200). Create: `{ok:true, mode:"create", job, state:"queued", name}`. Update of an existing managed stack: `{ok:true, mode:"update", job, state:"queued", name}` (job action `stack-update`). Heavy job |
| `POST /stacks/reconcile` | `stack-reconcile` | `{name, mode, nonce}` (`mode` is `adopt-live` or `re-render`) | `adopt-live` is sync: `{name, mode, spec, ssh_port}`. `re-render` is a job: `{ok, job, state, name, mode}` (job result like update plus `mode` and `dropped`). `404` `stack not found`. `409` `live files do not match a blueprint render; use re-render` (adopt-live) or `stack busy` |
| `POST /stacks/fix-permissions` | `stack-fix-permissions` | `{name, nonce}` | `{job, state}`. WordPress stacks under `/opt/apps/wordpress` only; `400` `"only wordpress stacks"` / `"stack folder not available"`. Heavy job |
| `POST /stacks/delete` | `stack-delete` | `{name, confirm, final_dump, nonce}` — `confirm` must equal `name`; `final_dump` boolean | `{ok:true, job, state:"queued", name}`. WordPress, **managed** stacks only. Heavy job. `400` `confirm must equal name` / `invalid final_dump` / `only wordpress stacks`; `409` `stack is not managed by stack-api` / `stack is already in the trash` / `stack busy` |
| `POST /stacks/trash/list` | `stack-trash-list` | `{nonce}` | `{retention_days, trash:[{name, type, created_by, deleted_at, purge_after, ssh_port, volumes, final_dump}]}`. Filtered by principal scope. `final_dump` is a path or null |
| `POST /stacks/trash/restore` | `stack-trash-restore` | `{name, nonce}` | `{ok:true, job, state:"queued", name}`. Heavy job. `404` `not in trash`; `409` `stack busy` (detail may be `trash folder missing`, `<path> already exists`, `volume <v> missing`, `ssh port <n> is in use`) |
| `POST /stacks/trash/purge` | `stack-trash-purge` | `{name, confirm, nonce}` — `confirm` must equal `name` | `{ok:true, job, state:"queued", name}`. Heavy job, per-stack lock. Scope like trash list/restore. Removes named volumes and the trash folder, sets `purged_at`, drops the stack from the owner map, frees the name. `400` `confirm must equal name` / `invalid name`; `404` `not in trash` (also when not visible); `409` `stack busy`. Job 409 `not in trash anymore` |

Both `plan` and `apply` refuse: names not visible to the principal (sandbox), unmanaged existing stacks / existing folder / existing Caddy site file, a name already in the trash (`name: <name> is in the trash until <ts>; restore it, purge it, or wait for the purge`). An existing **managed** stack is an update (`mode: "update"`), not a create. A sandbox principal may plan/apply any valid `<label>.staging.foxcraft.digital` name; after apply it owns that stack. PHP version cannot be changed via update. Drift blocks update until reconcile.

Docker errors → `502` `"docker error"`.

`ports` includes the bind IP (`ip`, `public`, `private`, `type`) — e.g. `0.0.0.0` and `::` for 2206.

FastAPI docs (`/docs`, `/redoc`, `/openapi.json`) are disabled.

## Spec (WordPress, v1)

Code: `app/stackspec.py` (`validate_spec`, people loader, `render_wordpress`, `render_caddy`, `policy`, `dump_compose`). `pyyaml` is in `requirements.txt`. Empty `php_ini` is a JSON object (`{}`), never `[]`.

| Key | Rule |
|---|---|
| `type` | only `"wordpress"` |
| `name` | exactly one label + `.staging.foxcraft.digital` (`STACK_API_DOMAINS`, wildcard cert covers one level). Label `^[a-z0-9]([a-z0-9-]{0,61}[a-z0-9])?$`, not `atlas` / `api` / `stella` / `edison` / `york` / `www` / `mail`, not `stella-*` |
| `php` | `STACK_API_PHP_VERSIONS` (live `.env` `8.4,8.5`; code default `8.5`) |
| `ssh_users` | list of person names from `people/` (max 20, unique; unknown → error). uid + key come from `people/`. Atlas requires at least one |
| `php_ini` | whitelist only; each value validated: `memory_limit`, `upload_max_filesize`, `post_max_size`, `max_execution_time`, `max_input_time`, `max_input_vars`, `display_errors`, `date.timezone`, `session.gc_maxlifetime`, `opcache.enable`, `opcache.memory_consumption`, `opcache.max_accelerated_files`, `opcache.validate_timestamps`, `opcache.revalidate_freq` |

## Policy (allowlist)

Checked on every rendered compose (plan and apply). Top-level keys only `services` / `networks` / `volumes`. Networks only `edge` (external) and `internal`. Named volumes without options. Service keys only `image`, `container_name`, `environment`, `volumes`, `depends_on`, `networks`, `ports`, `restart`, `pull_policy`. Images: `wordpress:php8.x-apache`, `mariadb:11`, `stella/ssh-php:8.x` (`pull_policy: never` only for `stella/`). On Stella: `wordpress:php8.4-apache` and `stella/ssh-php:8.4` (plus the existing 8.5 tags). `container_name` = `<name>` for `wp`, `<name>-<service>` otherwise. `restart: unless-stopped`. Only `wp` on `edge`. Only `ssh` publishes ports, `22100–22199:22`. Bind mounts only `./` paths without `..`. Env values are strings; newlines only in `WORDPRESS_CONFIG_EXTRA`.

## WordPress blueprint

Like hypershape:

| Service | Image | Networks | Notes |
|---|---|---|---|
| `wp` | `wordpress:php<v>-apache` | `edge` + `internal` | `WORDPRESS_CONFIG_EXTRA` with `$$`-escaped HTTPS fix + `WP_HOME` / `WP_SITEURL`. `WORDPRESS_TABLE_PREFIX: ${WP_TABLE_PREFIX:-wp_}` (from the stack `.env`; stacks without the variable keep `wp_`) |
| `db` | `mariadb:11` | `internal` | named volume `db-data` |
| `ssh` | `stella/ssh-php:<v>` | `internal` | `pull_policy: never`, port `221xx:22`, `./atlas:/etc/atlas:ro`, `home` + `ssh-keys` volumes, `./html:/home/<user>/app` per user |

`./php/zz-atlas.ini` is mounted read-only into `/usr/local/etc/php/conf.d/` in `wp` and `ssh`. No ssh users → no ssh service, no port, no `atlas/` files. Caddy site block has **no `tls` section** (wildcard cert); only `@blocked not remote_ip` from `STACK_API_SITE_ALLOW_IPS` and `reverse_proxy <name>:80`.

**Site allowlist for new stacks (decided 2026-11-08):** `194.126.177.181` (Proton), `78.47.218.190` (york), `2.31.42.13` (Atlas prod).

SSH ports for stack-api managed stacks: **22100–22199**, lowest free; a port freed by a deleted stack can be reused. One `DOCKER-USER` range rule — see [`infrastructure.md`](infrastructure.md) / [`dev-ssh-access.md`](dev-ssh-access.md). Manual ports stay 2201–2207 (next free manual: 2208).

`reconstruct` / `adopt-live` only matches stacks whose compose contains the `WORDPRESS_TABLE_PREFIX` line. Older stacks (sandbox) get a compose diff (that env line added) on their next update — harmless; the prefix stays `wp_`.

## Jobs

SQLite table `jobs` (`id`, `created_at`, `started_at`, `finished_at`, `principal`, `action`, `stack`, `service`, `state` `queued`/`running`/`ok`/`error`, `result` JSON, `detail`). One in-memory asyncio lock per stack: a second action on a busy stack (sync restart or any job) gets `409` `stack busy`. Heavy-job semaphore `STACK_API_HEAVY_JOBS` (default 1) — `apply`, `update`, `reconcile` (re-render), `people-sync`, `fix-permissions`, `delete`, `trash-restore`, `trash-purge`. On startup, queued/running jobs are set to `error` `interrupted by stack-api restart`. Job completion writes a second audit row `job=<id> done …`. Locks and the replay cache require a single uvicorn worker.

## `stack_state` (managed stacks)

SQLite table `stack_state` (`name` PK, `type`, `spec_json`, `rendered_hash`, `ssh_port`, `created_by`, `created_at`, `updated_at`, `deleted_at`, `trash_path`, `volumes_json`, `final_dump`, `purged_at`). A stack is **managed** when it has a live row (`deleted_at` and `purged_at` null). Only managed WordPress stacks can be deleted. Hand-built sites (`dominikliss`, `hypershape`) stay unmanaged until adopted.

**Startup backfill (2026-11-14):** existing WordPress stacks are adopted only when (1) a successful `stack-apply` job exists for that name and (2) a re-render of that spec matches the live files byte for byte. Adopted: `sandbox.staging.foxcraft.digital`, owner `atlas-dev`. `hypershape` (hand-edited compose) and `dominikliss` (no apply job) were not adopted.

A successful **apply**, **update**, or **reconcile** writes `stack_state` (`created_by` = the signing principal on create, `rendered_hash` of the files just written, `ssh_port`, `volumes_json`). `rendered_hash` is sha256 over `docker-compose.yml`, `php/zz-atlas.ini`, everything under `atlas/`, and the Caddy site file. **Drift** is `true` when the live hash ≠ `rendered_hash`, `false` when they match, `null` when the stack is not managed. Shown on `list` / `get`. Drift blocks plan/apply update until reconcile (`adopt-live` or `re-render`).

## Apply job

Order (create): files → compose-up → caddy → http-`<status>` → wp-install (or wp-already-installed) → permissions → state.

1. Pick SSH port (lowest free in 22100–22199, inside the heavy-job slot so two applies cannot race), render, policy.
2. Files: `mkdir /opt/apps/wordpress/<name>` (fails if it exists), `html/`, `.env` (`chmod 600`, `DB_PASSWORD` + `DB_ROOT_PASSWORD` random hex, never returned/logged; `WP_TABLE_PREFIX` = one lowercase letter + 4 lowercase letters/digits + `_`, e.g. `k3f9a_`), `php/zz-atlas.ini`, `atlas/users.conf` + keys, `docker-compose.yml`.
3. `docker compose up -d --quiet-pull` in the folder (project name from folder). Compose has `WORDPRESS_TABLE_PREFIX: ${WP_TABLE_PREFIX:-wp_}` on `wp`, so stacks without the variable (sandbox) keep `wp_`.
4. Write `/caddy-sites/<name>.caddy`, `docker exec services-caddy-1 caddy reload --config /etc/caddy/Caddyfile --adapter caddyfile` (a bad site file fails the reload; old config keeps running).
5. Wait up to 180 s for 200/301/302 from `curl http://localhost/` inside the wp container.
6. WordPress install in a throwaway container after the HTTP check. Image `stella/ssh-php:<php>`; `WORDPRESS_*` env copied from the wp container:

   ```bash
   docker run --rm --pull never --volumes-from <name> --network container:<name> --user 33:33 --entrypoint wp stella/ssh-php:<php>
   ```

   First `wp core is-installed`. If not installed: `wp core install --url=https://<name> --title=<first label of name> --admin_user=<STACK_API_WP_ADMIN_USER> --admin_email=<STACK_API_WP_ADMIN_EMAIL> --admin_password=<random token_urlsafe(24)> --skip-email`. Defaults: user `foxcraft.digital`, email `projects@foxcraft.digital`. The password appears only once, in the job result (`admin.password`), and is masked as `***` in error output. Each new stack gets a random table prefix (`WP_TABLE_PREFIX` in the stack `.env`, 1 letter + 4 alphanumerics + `_`), returned as `table_prefix`. Step name `wp-install`, or `wp-already-installed` when core was already there. Plan notes mention the install and the prefix. Install failure → step `wp-install` error → normal rollback (stack removed).
7. Fix permissions (below).
8. Write `stack_state` (`created_by` = principal).

On any failure: remove the site file + reload Caddy, `compose down -v --remove-orphans`, delete the folder (only if this job created it). Job result on error: `{step, output, done, rollback}`. Success result: `{name, path, url, ssh_port, ssh_users, steps, table_prefix, admin: {user, email, password}}`. `password` is `null` when WordPress was already installed. The password is stored in SQLite `jobs.result` and is **not** written to the audit log or the job `detail` string. Create responses also include `mode: "create"`.

**Incident 2026-11-14:** the first create with auto-install failed at step `wp-install` (`missing the MySQL extension`): `stella/ssh-php` was built on `php:<v>-cli` without `mysqli`. Rollback worked (Caddy site removed, compose down, folder removed). Fixed by adding `mysqli` + `pdo_mysql` to the image (see [Shared SSH image](#shared-ssh-image-stellassh-phpversion)).

## Update job

Same `POST /stacks/plan` / `POST /stacks/apply` as create. When `spec.name` is an existing **managed** stack the response is `mode: "update"` and the job action is `stack-update`.

Order:

1. Validate spec. Refuse a PHP change (`php: is 8.5; the PHP version is changed with the PHP version job, not with update`) and a type change (`type: cannot be changed`).
2. Drift check — live hash ≠ `rendered_hash` → `drift: live files differ from the last apply; reconcile first`.
3. Render + policy. Diff against live files (`changes:[{file, status: added|changed|removed, diff}]` on plan).
4. Backup current files to `<stack>/.atlas-bak/<UTC ts>/` (Caddy site file stored as `caddy.site`; last 10 backups kept; folder mode `0700`). Relative path in the job result (e.g. `.atlas-bak/20261114T…Z`).
5. Write **only changed files in place** (keeps inodes for single-file bind mounts) and remove `atlas/` files that are no longer rendered.
6. `docker compose up -d --remove-orphans` if compose changed.
7. `docker compose restart wp` (and `ssh` if present) if `php/zz-atlas.ini` changed.
8. `docker exec <name>-ssh atlas-sync-users` if `atlas/` changed (fallback: restart `ssh`).
9. Caddy reload if the site file changed.
10. HTTP check. Store spec + `rendered_hash`.

SSH port is kept. The first SSH user on a stack that had no `ssh` service gets the lowest free port in **22100–22199**. Removing all users removes the ssh service and frees the port. Atlas currently requires at least one SSH user.

Success: `{name, changed, changes:[{file, status}], backup, ssh_port, ssh_users[], steps[]}`. `changed: false` means no file changes (neutral, not an error). Failure: restore the backed-up files, `compose up`, restart, reload Caddy; result `{step, output, done[], rollback[], backup}` or the string `drift: …`.

## Reconcile

`POST /stacks/reconcile`, action `stack-reconcile`, `{name, mode, nonce}`.

- `adopt-live` (sync): reconstruct the spec from live files; they must match a blueprint render **byte for byte**. `{name, mode, spec, ssh_port}`. `409` `live files do not match a blueprint render; use re-render`.
- `re-render` (job): write from the stored spec (no drift check). People missing from `people/` are dropped. `{ok, job, state, name, mode}`. Job result like update plus `mode` and `dropped`. Backup kept.

`404` `stack not found`. `409` `stack busy`.

## People sync

`out_of_sync` (on `people/list` per managed stack) compares the rendered `atlas/` files with the live `atlas/` files. `POST /people/sync`, action `stack-people-sync`, `{name?, nonce}`: no `name` means every out-of-sync stack. Sync is an update with the stored spec and the current `people/` list (removed people are dropped). Response `{"jobs":[{name, job, state} | {name, error}]}`. Job result like update plus `dropped`.

## Delete job

Only **managed** WordPress stacks. `confirm` must equal `name`. Heavy job. Atlas currently sends `final_dump: false` (restore uses volumes); when `final_dump` is true the job writes a gzipped SQL dump first.

Order:

1. If `final_dump`: dump MariaDB to `dumps/<ts>-final.sql.gz` inside the stack folder (path stored on `stack_state.final_dump`).
2. `docker compose down --remove-orphans` **without** `-v` (named volumes kept).
3. Remove `/caddy-sites/<name>.caddy` and reload Caddy.
4. Move the folder to `/opt/apps/.trash/<name>-<ts>`. Set `deleted_at`, `trash_path`, `volumes_json`.

Success result: `{name, trash, volumes, final_dump, deleted_at, purge_after, steps}`. On failure: `{step, output, done, rollback}`. Unmanaged stacks (`hypershape`, `dominikliss`) get `409` `stack is not managed by stack-api`.

## Trash list / restore / purge

`stack-trash-list` returns `{retention_days, trash:[…]}` for rows with `deleted_at` set and `purged_at` null, filtered by principal scope. `retention_days` is `STACK_API_TRASH_DAYS` (live `.env` `7`).

**Restore** (heavy job): move the folder back from `trash_path`, `compose up -d`, write the Caddy site file + reload, clear `deleted_at` / `trash_path`. Volumes were never removed on delete. `404` `not in trash`. `409` when the trash folder is missing, the live path already exists, a recorded volume is gone, or the SSH port is in use.

**Purge now** and the **hourly purge** share one function — see [Purge job](#purge-job).

## Purge job

Implemented 2026-11-14 (`app/main.py.bak-20261114c` is the backup from before purge-now). Atlas `POST /api/stack-trash/{name}/purge`.

`POST /stacks/trash/purge`, action `stack-trash-purge`, `{name, confirm, nonce}`. `confirm` must equal `name`. Heavy job, per-stack lock, `STACK_API_HEAVY_JOBS` semaphore. Scope: same as trash list / restore (sandbox sees allowlist ∪ owned). Auth: IP + token + signature, unchanged.

Immediate response: `{ok:true, job, state:"queued", name}`.

Row must already be in trash: `deleted_at` set, `purged_at` null. Otherwise `404` `not in trash` (also when the name is not visible). `400` `confirm must equal name` / `invalid name`. `409` `stack busy`. If the row leaves trash before the job runs: job 409 `not in trash anymore`.

Job:

1. Remove each volume in `volumes_json` (`docker volume rm`). HTTP 404 / already gone is fine.
2. Remove the trash folder (`trash_path`). Missing folder is fine (step `trash-folder-missing`).
3. Set `purged_at` (UTC ISO). Drop the stack from the owner map. `plan` / `apply` treat a purged row as absent (name is free). Do not reuse the row for a later create — a new create inserts a fresh live row.
4. Audit: job action `stack-trash-purge` with the signing principal. The hourly purge uses the same function, audited as principal `system` (action `stack-purge`).

Success `jobs.result`: `{name, volumes, trash, purged_at, steps}`. Failure: `{step:"purge", output, done}` — no rollback, purge is one-way.

After purge:

- `stack-trash-list` omits it
- `plan` / `apply` treat the name as a new create
- the old SSH port is free
- `jobs/get`: a sandbox principal can still read jobs of stacks it created after they were purged (needed to poll the purge job)

The hourly purge runs after `STACK_API_TRASH_DAYS` (default 7) as principal `system`, action `stack-purge`.

## Permissions (apply step + `fix-permissions` action)

On `html/`: `chown -R 33:33` (www-data), `chmod -R g+rwX`, SetGID on all dirs, `setfacl -R -m g:33:rwX,d:g:33:rwX`. SSH users are in www-data (`atlas-sync-users`), so they can write, and files WordPress creates later stay group-writable. Verified 2026-11-08: `dominik` on sandbox port 22100 → `drwxrwsr-x+`, write OK.

## Audit log

SQLite table `audit` (`id`, `ts` UTC ISO, `principal`, `action`, `stack`, `service`, `result` `ok`/`error`, `detail`). Triggers block `UPDATE` and `DELETE` (`audit is append-only`, verified). Protects against API/SQL changes, not against root on Stella.

Every signed action goes through one helper (`run_action`). Result is `ok` or `error`, including `400` `invalid name`, `404` `stack not found` / `service not found`, and `502` Docker errors. For invalid input the raw value is stored (cut to 200 chars), e.g. stack `../services`. `detail` is cut to 2000 chars. If the audit write fails, it is logged to stdout and the request continues.

Rejected auth is audited with principal `-`:

| Action | `detail` | Examples |
|---|---|---|
| `denied` | `reason=<reason> path=<path> ip=<ip>` | bad token, bad signature, replay, stale timestamp |
| `ip-denied` | `ip=<ip> path=<path>` | missing / disallowed `X-Forwarded-For` |

```bash
docker exec stack-api python -c "import sqlite3; [print(r) for r in sqlite3.connect('/data/stack-api.sqlite').execute('SELECT id, ts, principal, action, stack, service, result FROM audit')]"
```

---

## DNS and certificate

- A record `stella-stack-api.foxcraft.digital` → `95.217.144.93` (TTL 300), created 2026-11-04 via `POST /v1/zones/567656/rrsets`.
- Caddy block with the `dir` pin plus `propagation_delay 45s` / `propagation_timeout 3m` (same as `stella-media-api`). First attempt succeeded 2026-11-04, issuer `acme-v02.api.letsencrypt.org-directory` (production).

```caddyfile
stella-stack-api.foxcraft.digital {
    tls {
        issuer acme {
            email projects@foxcraft.digital
            dir https://acme-v02.api.letsencrypt.org/directory
            dns hetzner {env.HETZNER_API_TOKEN}
            propagation_delay 45s
            propagation_timeout 3m
        }
    }
    @blocked not remote_ip 2.31.42.13 95.217.144.93 172.20.0.1
    respond @blocked 403
    reverse_proxy stack-api:8000
}
```

---

## Verification (2026-11-04)

| Test | From | Result |
|---|---|---|
| `GET /health` | MacBook (`194.126.177.181`) | 403 (Caddy) |
| `GET /health` | Stella, Atlas prod | `{"status":"ok"}` |
| `GET /health` without `X-Forwarded-For` | inside `services-caddy-1` → `stack-api:8000` | 403, log `denied ip=None` |
| `GET /health` with fake `X-Forwarded-For: 1.2.3.4` | inside `services-caddy-1` | 403, log `denied ip=1.2.3.4` |
| Signed `POST /ping` | Atlas prod | `ok`, `principal: atlas-prod` |
| Same request again | Atlas prod | 401, log `denied reason=replay` |
| Wrong token | Atlas prod | 401, log `denied reason=bad token` |
| `POST /stacks/list` | Atlas prod | all 6 stacks (`advoapp.finditoo`, `atlas.dev`, `dominikliss.foxcraft.digital`, `edison.dev`, `hypershape`, `osgar.datahub`); no `/opt/services` containers |
| `POST /stacks/get` `hypershape` | Atlas prod | `wp` / `db` / `ssh` with `started_at` / `restart_count` |
| `POST /stacks/restart` hypershape `ssh` | Atlas prod | `ok` |
| name `../services` | Atlas prod | `400` `invalid name`; audit `error` row with stack `../services` |
| name `services` | Atlas prod | `404` `stack not found`; audit `error` row |
| restart service `nope` | Atlas prod | `404`; audit `error` row |
| wrong token on `/stacks/list` | Atlas prod | `401`; audit row (`-`, `denied`, `reason=bad token path=/stacks/list ip=2.31.42.13`) |
| `ports` on list/get | Atlas prod | bind IP present (`0.0.0.0` and `::` for 2206) |
| audit after the first list/get/restart | Stella | 3 `ok` rows; `DELETE FROM audit` failed with `audit is append-only` |
| atlas.dev app → public URL | inside atlas.dev container | Caddy peer `172.20.0.1`; after allowlist, principal `atlas-dev` lists only sandbox `dominikliss.foxcraft.digital` |
| Identical request in the same second (no nonce) | Atlas | 401 `replay` (01:29, audit row 19) — Ed25519 signatures are deterministic |

Test helper on host `atlas` (token from Atlas `.env`, sign with `/home/atlas/.ssh/atlas-stack-api`, namespace `stack`):

```bash
stack_call() {
  local ACTION="$1" URLPATH="$2" REQ="$3"
  local STACK_TOKEN TS HASH PAYLOAD TMP BODY
  STACK_TOKEN=$(grep '^STELLA_STACK_API_TOKEN=' /var/www/atlas/.env | cut -d= -f2- | tr -d '[:space:]')
  REQ=$(python3 -c 'import json,os,sys; d=json.loads(sys.argv[1]); d.setdefault("nonce", os.urandom(16).hex()); print(json.dumps(d,separators=(",",":")))' "$REQ")
  TS=$(date +%s)
  HASH=$(printf %s "$REQ" | sha256sum | cut -d' ' -f1)
  PAYLOAD="${ACTION}:${TS}:${HASH}"
  TMP=$(sudo -u atlas mktemp -d)
  printf %s "$PAYLOAD" | sudo -u atlas tee "$TMP/payload" >/dev/null
  sudo -u atlas ssh-keygen -Y sign -f /home/atlas/.ssh/atlas-stack-api -n stack "$TMP/payload"
  BODY=$(python3 -c 'import json,sys; print(json.dumps({"payload":sys.argv[1],"signature":open(sys.argv[2]).read(),"request":sys.argv[3]}))' "$PAYLOAD" "$TMP/payload.sig" "$REQ")
  curl -s -X POST "https://stella-stack-api.foxcraft.digital${URLPATH}" \
    -H "Content-Type: application/json" \
    -H "X-Stack-Token: $STACK_TOKEN" \
    -d "$BODY"
  echo
  sudo -u atlas rm -rf "$TMP"
  unset STACK_TOKEN
}

# stack_call stack-list /stacks/list '{}'
# stack_call stack-get /stacks/get '{"name":"hypershape"}'
# stack_call stack-restart /stacks/restart '{"name":"hypershape","service":"ssh"}'
```

One-shot ping (same signing; prefer `stack_call` for the stack endpoints):

```bash
STACK_TOKEN=$(grep '^STELLA_STACK_API_TOKEN=' /var/www/atlas/.env | cut -d= -f2- | tr -d '[:space:]')
REQ=$(python3 -c 'import json,os; print(json.dumps({"hello":"stella","nonce":os.urandom(16).hex()}))')
TS=$(date +%s)
HASH=$(printf %s "$REQ" | sha256sum | cut -d' ' -f1)
PAYLOAD="stack-ping:${TS}:${HASH}"
TMP=$(sudo -u atlas mktemp -d)
printf %s "$PAYLOAD" | sudo -u atlas tee "$TMP/payload" >/dev/null
sudo -u atlas ssh-keygen -Y sign -f /home/atlas/.ssh/atlas-stack-api -n stack "$TMP/payload"
BODY=$(python3 -c 'import json,sys; print(json.dumps({"payload":sys.argv[1],"signature":open(sys.argv[2]).read(),"request":sys.argv[3]}))' "$PAYLOAD" "$TMP/payload.sig" "$REQ")
curl -s -X POST https://stella-stack-api.foxcraft.digital/ping -H "Content-Type: application/json" -H "X-Stack-Token: $STACK_TOKEN" -d "$BODY"; echo
sudo -u atlas rm -rf "$TMP"; unset STACK_TOKEN
```

---

## Verification (2026-11-08)

atlas.dev, principal `atlas-dev`. First created stack: `sandbox.staging.foxcraft.digital`. Backups on Stella: `app/main.py.bak-20261108`, `-20261108b`, `-20261108c`, `-20261108d`; `app/stackspec.py.bak-20261108`, `-20261108b`, `-20261108c`; `Dockerfile.bak-20261108`.

| Test | From | Result |
|---|---|---|
| Two `list` / `get` calls in the same second | atlas.dev | both ok (nonce) |
| `stack-logs` `wp` / `db` / `ssh` `lines=200` | atlas.dev | ok |
| Async restart of `ssh` | atlas.dev | job `ok` |
| plan for `sandbox.staging.foxcraft.digital` | atlas.dev | ok, port 22100, compose wp/db/ssh, Caddy without tls |
| plan for `sandbox.foxcraft.digital` | atlas.dev | name error |
| plan for `dominikliss.foxcraft.digital` | atlas.dev | `stack exists` |
| apply `sandbox.staging.foxcraft.digital` (`dominik`, `pawel`, `edison`) | atlas.dev | job `ok`; `https://sandbox.staging.foxcraft.digital` shows the WP installer (only from allowlisted IPs) |
| SSH `dominik@95.217.144.93:22100` | MacBook | uid 1000, groups www-data + atlasusers; write OK after fix-permissions |

---

## Changes (2026-11-14)

`main.py` only. `stack_state` + delete / trash / restore / hourly purge. Startup backfill adopted `sandbox.staging.foxcraft.digital` (owner `atlas-dev`). Compose: `/opt/apps:/opt/apps` and `./stack-api/people:/app/people:ro`. `.env`: `STACK_API_TRASH_DAYS=7`, `STACK_API_PHP_VERSIONS=8.4,8.5`. Images on Stella: `stella/ssh-php:8.4`, `wordpress:php8.4-apache` (8.5 tags unchanged).

Update / drift / people sync: `list` / `get` return `drift`; `get` also returns `spec` and `ssh_port`. `plan` / `apply` on an existing managed stack use `mode: "update"` (job action `stack-update`). `POST /stacks/reconcile` (`adopt-live` / `re-render`). `people/list` adds per-stack `out_of_sync` / `dropped` / `errors`. `POST /people/sync`. WordPress auto-install on create: random `WP_TABLE_PREFIX` in the stack `.env`, WP-CLI install via a one-off `stella/ssh-php` container, job result `table_prefix` + `admin` (password once in `jobs.result`, not in audit/detail). Trash purge-now: `POST /stacks/trash/purge` (`stack-trash-purge`); hourly purge uses the same function (`stack-purge`, principal `system`). PHP version change, labels, basic auth, staging guard, cron still open.

## Verification (2026-11-14)

| Test | From | Result |
|---|---|---|
| `atlas-sync-users` path in `stella/ssh-php` | Stella | `/usr/local/bin/atlas-sync-users` |
| stack-api startup | Stella | log `managed stacks: sandbox.staging.foxcraft.digital (atlas-dev)` — update code is in the running container |
| Edit `sandbox.staging.foxcraft.digital` `memory_limit` 512M Plan → Apply | atlas.dev | **no request** — Atlas did not send `stack-plan` / `stack-apply` (see [open-gaps](../open-gaps.md#stack-api-atlas-container-management--2026-11-04)) |
| update end to end | — | **UNVERIFIED** |
| reconcile (`adopt-live` / `re-render`) | — | **UNVERIFIED** |
| people sync | — | **UNVERIFIED** |
| trash purge after `STACK_API_TRASH_DAYS` | — | **UNVERIFIED** |
| `php -m` lists `mysqli` and `pdo_mysql` in `stella/ssh-php:8.4` and `:8.5` | Stella | verified 2026-11-14 (images rebuilt after Dockerfile `docker-php-ext-install mysqli pdo_mysql`) |
| trash purge-now (`POST /stacks/trash/purge`) | atlas.dev | **UNVERIFIED** |
| create with WordPress auto-install after the mysqli fix | atlas.dev | **UNVERIFIED** |

---

## Shared SSH image (`stella/ssh-php:<version>`)

Built 2026-11-04. One image per PHP version, used by WordPress stacks' `ssh` service instead of a per-site `Dockerfile.ssh`. First stack: `hypershape`. Pattern: [`wordpress-staging.md`](wordpress-staging.md#ssh-container-shared-image). Updates: [`php-updates.md`](php-updates.md#shared-ssh-image-stellassh-php).

**Source:** `/opt/services/stack-api/images/ssh-php/` — `Dockerfile` (`ARG PHP_VERSION` default `8.5`, `FROM php:${PHP_VERSION}-cli`), `atlas-sync-users`, `entrypoint.sh`.

**Build** (local image only; compose uses `pull_policy: never`):

```bash
docker build --build-arg PHP_VERSION=8.5 -t stella/ssh-php:8.5 /opt/services/stack-api/images/ssh-php
docker build --build-arg PHP_VERSION=8.4 -t stella/ssh-php:8.4 /opt/services/stack-api/images/ssh-php
```

On Stella: `stella/ssh-php:8.4` and `stella/ssh-php:8.5`; WordPress images `wordpress:php8.4-apache` and `wordpress:php8.5-apache`.

**Contents:** `openssh-server`, `git`, `curl`, `wget`, `unzip`, `less`, `procps`, `mariadb-client`, WP-CLI phar at `/usr/local/bin/wp`, group `atlasusers`. Dockerfile has `RUN docker-php-ext-install mysqli pdo_mysql` after the apt layer (backup `images/ssh-php/Dockerfile.bak-20261114`). Verified on 8.5: PHP 8.5.12, WP-CLI 2.12.0, MariaDB client 11.8.6. Rebuilt `stella/ssh-php:8.5` and `:8.4` on 2026-11-14; verified `php -m` lists `mysqli` and `pdo_mysql` in both. Running ssh containers (`hypershape`, `sandbox.staging.foxcraft.digital`, …) still use the old image until recreated; only needed if SSH users run `wp db` commands there. Throwaway install containers pick up the rebuilt image immediately.

**Known noise:** every `wp` command on PHP 8.5 prints a `Deprecated: Case statements followed by a semicolon` notice from WP-CLI's bundled `react/promise`. Harmless, not suppressed.

### User sync

`/etc/atlas` is mounted read-only with `users.conf` (`name uid` per line, `#` comments) and `keys/<name>.pub`. `atlas-sync-users` validates name (`^[a-z][a-z0-9_-]{0,31}$`) and uid (1000–59999), creates missing users (`useradd -u <uid> -U -M`), refuses UID mismatches, adds users to `www-data` + `atlasusers`, sets `/home/<name>` `750`, `.ssh` `700`, copies the key to `authorized_keys` `600` (removes it if no key file), and removes `atlasusers` members not in `users.conf` (kills processes, `userdel -f`, home kept in the volume).

Live run without restart: `docker exec <stack>-ssh atlas-sync-users`. Keep UIDs stable (existing WP stacks: `dominik` 1000, `pawel` 1001, `edison` 1002).

### `entrypoint.sh`

Creates `/run/sshd` and `/etc/ssh/sshd_config.d`, `ssh-keygen -A` (missing host keys only; `ssh-keys` volume keeps existing ones), writes `/etc/ssh/sshd_config.d/atlas.conf` (`PermitRootLogin no`, `PasswordAuthentication no`, `KbdInteractiveAuthentication no`, `AllowGroups atlasusers`), runs `atlas-sync-users`, then `exec /usr/sbin/sshd -D -e`. Port forwarding stays enabled (Cursor Remote-SSH).

### Verification (2026-11-04, hypershape)

| Test | Result |
|---|---|
| Start log | added `dominik` / `pawel` / `edison` + `Server listening` |
| MacBook login as `dominik` on 2206 | OK, no host-key warning; write test OK; `wp core version` 7.1.3 |
| Live add `testuser` 1003 then remove | `removed testuser (home kept)` |
| File in `/home/dominik` | survives `docker compose up -d --force-recreate ssh` |
| `getent hosts db` from `-ssh` | resolves |

## Roadmap

Single list. Do not duplicate it in [`../open-gaps.md`](../open-gaps.md).

### Foundation

- [x] Auth (IP Caddy+app, token, signature namespace `stack`, replay), DNS/cert
- [x] `list` / `get` / `restart`, append-only audit log incl. rejected requests
- [x] Principals + sandbox scope (`PRINCIPAL_SCOPES`, `STACK_API_SANDBOX_STACKS`; unknown principals rejected)
- [x] Shared SSH image `stella/ssh-php` (`users.conf` + `keys/`, live sync, persistent `/home`, WP-CLI)
- [x] Job table in SQLite for long-running actions (one lock per stack, limit for heavy jobs) (2026-11-08)
- [x] `logs` endpoint (last N lines per service) (2026-11-08)
- [x] `plan` dry-run (render compose / files / Caddy; no apply) (2026-11-08)
- [x] `apply` create job (2026-11-08) — first stack `sandbox.staging.foxcraft.digital`
- [x] people list (`people/` bind-mounted; spec takes person names)
- [x] `fix-permissions` action (2026-11-08)
- [x] `stack_state` + startup backfill of byte-identical apply jobs (2026-11-14); adopted `sandbox.staging.foxcraft.digital`

### apply (create/change from a spec)

- [x] Spec schema with strict validation (names, PHP versions from a list, whitelisted ini keys, no newlines; paths must resolve under `/opt/apps/<type>/<name>/`) (2026-11-08)
- [x] Policy check on every rendered compose before `up` (refuse `privileged`, `cap_add`, `network_mode: host`, `pid: host`, bind mounts outside the app folder, `docker.sock`, `edge` on `ssh`/`db`/`adminer`, images outside the allowlist) (2026-11-08)
- [x] WordPress blueprint (template of the hypershape compose; `ssh` on `internal`) (2026-11-08)
- [x] SSH users via spec (now via `people/`; uid + key from the bind-mounted list, then `atlas-sync-users`) (2026-11-08)
- [x] People sync to existing stacks (`POST /people/sync`; a key change in `people/` still needs a sync or reconcile re-render) (2026-11-14)
- [ ] Outbound SSH key per user (`ssh-keys/generate`, returns only the `.pub`, for GitHub deploy keys)
- [ ] PHP version change as a job (DB dump first, `wp` + `ssh` image together, verify `php -v` / HTTP / fatals / ACL, revert)
- [ ] PHP settings: `PHP_INI_SCAN_DIR` with shared `php-defaults.ini` + per-site `zz-atlas.ini`, whitelisted keys (`memory_limit`, `upload_max_filesize`, `post_max_size`, `max_execution_time`, `max_input_time`, `max_input_vars`, `display_errors`, `date.timezone`, `session.gc_maxlifetime`, selected `opcache.*`); never `auto_prepend_file`, `disable_functions`, `open_basedir`, `extension`; return effective values via `php -i`
- [x] wildcard DNS + wildcard cert for `*.staging.foxcraft.digital` (2026-11-08); apply writes `/caddy-sites/<name>.caddy` and reloads Caddy
- [x] stack-api SSH ports 22100–22199, lowest free, one `DOCKER-USER` range rule (2026-11-08). A port freed by a deleted stack can be reused.
- [x] Drift detection (hash of rendered files; mismatch blocks update until reconcile). `list` / `get` return `drift`: `true` / `false` / `null` (2026-11-14)
- [x] Update existing managed stacks (`plan` / `apply` `mode: "update"`; PHP version is a separate job) (2026-11-14)
- [x] Reconcile: `adopt-live` (sync) or `re-render` (job) (2026-11-14)
- [ ] Adopt remaining unmanaged stacks: hypershape (done by hand, not in `stack_state`) → `dominikliss.foxcraft.digital` → `osgar`, `advoapp` (need .NET blueprint) → `atlas.dev`, `edison.dev` (need Laravel blueprint)

### WordPress extras

- [x] `fix-permissions` action (chmod/SetGID/setfacl after first start or updates) (2026-11-08)
- [x] WordPress install on create (random prefix, admin foxcraft.digital, projects@foxcraft.digital, title = subdomain label) (2026-11-14)
- [ ] Staging guard toggle (mu-plugin: no outgoing mail, noindex, `blog_public=0`)
- [ ] Real cron (`DISABLE_WP_CRON` + scheduled `wp cron event run`)
- [ ] DB dump / import / safe search-replace via WP-CLI
- [ ] Optional basic auth / IP restriction per site in the Caddy block
- [ ] Labels as toggles: `backup.exclude`, `health.enable`
- [ ] Git repos for custom plugins/themes; Edison workspace registration (`/v1/workspaces/ensure`)

### restore / delete / databases

- [ ] `restore`: staging from an Atlas backup as create + restore job (reuse the wp-staging-api code: manifest check, streaming decrypt, collation mapping, search-replace, deactivate mail plugins); also "refresh from newer backup"
- [x] `delete` / trash / restore (2026-11-14): soft delete to `/opt/apps/.trash/<name>-<ts>` (Caddy site removed, `compose down` without `-v`), optional `dumps/<ts>-final.sql.gz`, hourly purge after `STACK_API_TRASH_DAYS` (default 7) removes volumes + folder (audit principal `system`, action `stack-purge`)
- [x] trash purge now (2026-11-14): `POST /stacks/trash/purge` (`stack-trash-purge`, `{name, confirm}`). Same function as the hourly purge. Spec: [Purge job](#purge-job).
- [ ] Adminer on demand per stack (internal network only, one-time auto-login URL, TTL ~30 min, read-only DB user by default on non-sandbox stacks, dump before write access)

### Other stack types

- [ ] Laravel blueprint (`app`, queue, scheduler, `db` mysql 8.4, `ssh` with composer/node; version = build arg; `queue:restart` after changes)
- [ ] .NET blueprint (`-dev` with `dotnet watch` + supervisord `stopasgroup`/`killasgroup`, MSSQL named volume, `ssh` with node/supervisorctl/sqlcmd)

### Hardening / later

- [ ] Optional second confirmation (email/TOTP) for destructive actions on non-sandbox stacks
- [ ] Audit mirror in Atlas
- [ ] WP-CLI Deprecated notice on PHP 8.5 (suppress `E_DEPRECATED` for the `wp` wrapper only?)
