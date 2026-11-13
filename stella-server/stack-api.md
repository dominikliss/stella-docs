# stack-api (container management for Atlas)

**Status:** live since 2026-11-04. As of 2026-11-13: signed `list` / `get` / `restart` (sync + async) / `logs` / `plan` / `apply` / `fix-permissions` / people list, job table with per-stack locks, append-only audit log. Atlas UI on atlas.dev: [`../atlas/stacks.md`](../atlas/stacks.md) (list/search, logs, restart, WordPress Plan → Create, fix-permissions, jobs, history). First stack created end to end from Atlas on 2026-11-08: `sandbox.staging.foxcraft.digital`. Not yet: update of existing stacks, delete, restore, Laravel/.NET blueprints.

**URL:** `https://stella-stack-api.foxcraft.digital`

**Purpose:** the single entry point through which Atlas creates, changes, restarts and deletes app stacks (WordPress, Laravel, .NET) on Stella. Atlas holds the spec; stack-api renders compose files from its own blueprints and applies them. Staging from an Atlas backup becomes a `create` + `restore` job here; [`wp-staging-api.md`](wp-staging-api.md) is superseded and will not be deployed as a separate service.

Infra: [`infrastructure.md`](infrastructure.md). System map: [`../integration/system-overview.md`](../integration/system-overview.md).

**Rule (decided 2026-11-04):** stacks under `/opt/apps/` are not edited by hand. Changes go through Atlas → stack-api. Blueprint changes go through the templates in the repo.

---

## Files on Stella

```
/opt/services/stack-api/
  ├── Dockerfile          ← multi-stage: docker CLI from docker:cli, compose plugin from docker/compose-bin:latest (→ /usr/libexec/docker/cli-plugins/docker-compose), python:3.13-slim + openssh-client + acl, ENV DOCKER_CONFIG=/tmp/docker (container is read-only), COPY people/ and COPY app/. Compose in the container: v5.7.0
  ├── requirements.txt    ← fastapi, uvicorn, httpx, pyyaml
  ├── .dockerignore       ← .env, allowed_signers, data, images
  ├── .env                ← chmod 600; see keys below
  ├── allowed_signers     ← mounted read-only, not baked into the image; atlas-prod + atlas-dev, both namespaces="stack"
  ├── data/               ← host dir; SQLite at /data/stack-api.sqlite in the container (code default; STACK_API_DB is not in .env)
  ├── people/
  │     ├── users.conf    ← central SSH people list (public keys only)
  │     └── keys/<name>.pub
  ├── app/main.py         ← API, auth, jobs, apply, rollback, permissions
  ├── app/stackspec.py    ← spec validation, people loader, WordPress blueprint, Caddy block, policy, YAML dump
  └── images/ssh-php/     ← shared SSH image (not in the API container)
        ├── Dockerfile
        ├── atlas-sync-users
        └── entrypoint.sh
```

Never write the token value into docs. Backups on Stella: `app/main.py.bak-20261108`, `-20261108b`, `-20261108c`, `-20261108d`; `app/stackspec.py.bak-20261108`, `-20261108b`, `-20261108c`; `Dockerfile.bak-20261108`.

`.env` keys (no values): `STACK_API_TOKEN`, `STACK_API_ALLOWED_IPS`, `STACK_API_SANDBOX_STACKS`, `STACK_API_SITE_ALLOW_IPS`, `STACK_API_DOMAINS`. `STACK_API_DB` is **not** in `.env` — the code default `/data/stack-api.sqlite` is used. Optional with defaults: `STACK_API_PHP_VERSIONS` (`8.5`), `STACK_API_HEAVY_JOBS` (`1`), `STACK_API_CADDY_CONTAINER` (`services-caddy-1`), `STACK_API_CADDY_SITES` (`/caddy-sites`), `STACK_API_PEOPLE` (`/app/people`).

`people/users.conf` + `people/keys/<name>.pub` are baked into the image. Current: `dominik` 1000, `pawel` 1001, `edison` 1002 (copied from hypershape 2026-11-08). Changing a key = edit `people/` + `docker compose up -d --build stack-api`. Existing stacks are not updated automatically yet.

Values: `STACK_API_SANDBOX_STACKS` = `dominikliss.foxcraft.digital,sandbox.staging.foxcraft.digital`. `STACK_API_SITE_ALLOW_IPS` = `194.126.177.181` (Proton), `78.47.218.190` (york), `2.31.42.13` (Atlas prod), decided 2026-11-08. `STACK_API_DOMAINS` = `staging.foxcraft.digital`.

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
      - ./stack-api/data:/data
      - /var/run/docker.sock:/var/run/docker.sock
      - /opt/apps/wordpress:/opt/apps/wordpress
      - ./caddy/sites:/caddy-sites
    read_only: true
    tmpfs:
      - /tmp
    networks:
      - stack
    restart: unless-stopped
```

`/opt/apps/wordpress` is the same path inside and outside — required: compose resolves `./` paths in the container, the daemon mounts them on the host, and the working_dir label must match stack discovery.

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

`STACK_API_SANDBOX_STACKS` in stack-api `.env` is the sandbox allowlist — currently `dominikliss.foxcraft.digital,sandbox.staging.foxcraft.digital` (the stack name is the folder name, not `dominikliss`). `atlas-dev` sees only those stacks; any other name is `404`. A sandbox principal may only **plan** / **apply** names in that list. `list` / `get` include `"sandbox"` on each stack; `ping` includes `"scope"`.

**Decision (2026-11-04):** `dominikliss.foxcraft.digital` is the sandbox. Re-decide before `delete`. **2026-11-08:** first stack created end to end from Atlas: `sandbox.staging.foxcraft.digital`. New stacks are `<label>.staging.foxcraft.digital`.

**Incident 00:51 (2026-11-04):** the `atlas-dev` signer was added to `allowed_signers` before the sandbox-restricting code was in the running container. atlas.dev then restarted the whole `osgar` stack (audit row 11). No damage. **Rule:** never add a principal before the restricting code is verified in the running container.

### Atlas `.env`

```
STELLA_STACK_API_URL=https://stella-stack-api.foxcraft.digital
STELLA_STACK_API_TOKEN=<value>
STELLA_STACK_API_ENABLED=false
STELLA_STACK_API_KEY_PATH=
```

Atlas client (2026-11-04, logs/jobs/nonce/plan/apply/people/fix-permissions 2026-11-08): `app/Domain/Stacks/StackApiClient.php`. Flag default **false** — `/api/stacks` is 404 and the nav item is hidden. Setting `stack_api_ssh_key_path` (Settings → GitHub, next to the deploy key path) overrides `STELLA_STACK_API_KEY_PATH`.

| Instance | Enable | Key path |
|---|---|---|
| atlas.dev | **on** 2026-11-04 (`STELLA_STACK_API_ENABLED=true` in `src/.env`) | `/var/www/.ssh/id_ed25519` |
| Atlas prod | **off** (flag stays false) | `/home/atlas/.ssh/atlas-stack-api` when enabled |

Admin role only. Atlas records restarts, apply, and fix-permissions in `stack_actions` (`user_id`, stack, service, result, detail, created_at`). Queued rows store the job id in `detail` and settle to `ok` / `error` when the job is fetched. stack-api keeps its own audit log.

### Atlas client

`StellaSshSigner::sign($payload, 'stack', $keyPath)` — namespace **`stack`**, not `deploy`. Payload `<action>:<unix_ts>:<sha256(request_json)>`. POST `{payload, signature, request}` where `request` is the exact JSON string that was hashed (encoded once; integers stay integers, booleans stay booleans). Header `X-Stack-Token`. Restart timeout 65 s (15 s when `async`). Logs timeout 30 s. Plan timeout 30 s. Apply timeout 30 s. People list and fix-permissions 15 s. A random `nonce` is sent on every request. Empty `spec` / `php_ini` are JSON objects (`{}`), never `[]`.

| Atlas | stack-api |
|---|---|
| `GET /api/stacks` | `POST /stacks/list` action `stack-list` request `{nonce}` |
| `GET /api/stacks/{name}` | `POST /stacks/get` action `stack-get` request `{"name", nonce}` |
| `GET /api/stacks/{name}/logs` | `POST /stacks/logs` action `stack-logs` request `{"name","service","lines"?, nonce}` (`lines` 1–1000, default 200) |
| `POST /api/stacks/{name}/restart` | `POST /stacks/restart` action `stack-restart` request `{"name", "service"?, "async"?, nonce}` — `async` only when true |
| `POST /api/stacks/{name}/fix-permissions` | `POST /stacks/fix-permissions` action `stack-fix-permissions` request `{"name", nonce}` |
| `GET /api/stack-jobs` | `POST /jobs/list` action `job-list` request `{"name"?, "limit"?, nonce}` (`limit` 1–100, default 20) |
| `GET /api/stack-jobs/{id}` | `POST /jobs/get` action `job-get` request `{"id", nonce}` |
| `GET /api/stack-people` | `POST /people/list` action `people-list` request `{nonce}` |
| `POST /api/stack-plan` | `POST /stacks/plan` action `stack-plan` request `{"spec", nonce}` (dry-run; no `stack_actions` row) |
| `POST /api/stack-apply` | `POST /stacks/apply` action `stack-apply` request `{"spec", nonce}` |

Use `/api/stack-jobs`, not `/api/stacks/jobs` — `jobs` would match `{name}`. Use `/api/stack-plan` / `/api/stack-people` / `/api/stack-apply`, not `/api/stacks/plan` and so on. Name `^[a-z0-9][a-z0-9.-]{0,252}$`, service `^[a-z0-9][a-z0-9_-]{0,62}$`, job id `^[0-9a-f]{32}$`, lines 1–1000, limit 1–100 — validated in Atlas before the call. Plan names are not pre-validated so stack-api can return sandbox / reserved-name errors. Mapped errors: 401/403 → `stack-api rejected the request`; 404 detail passed through; 400 detail passed through (`invalid input` when there is no detail); 409 → `stack busy`; 502 → `docker error` (results passed through); connection → `stack-api unreachable`. A handled plan or apply rejection is always HTTP 200 (`ok` + `errors`; `ok` can be false with a rendered compose). Never log the token, signature, key contents, log lines, spec keys, or response files. Async restart, apply, and fix-permissions record `stack_actions` with result `queued` and the job id in `detail` (`service` is the action for apply / fix-permissions). `plan` and `people` do not write `stack_actions`. Atlas `ssh_users` is a required list of person names (min 1).

---

## Stack discovery

A container belongs to a stack if its label `com.docker.compose.project.working_dir` is exactly `/opt/apps/<type>/<name>` with `type` in `{wordpress, laravel, dotnet}` and `name` matching `^[a-z0-9][a-z0-9.-]{0,252}$`. Stack name = folder name, type = parent folder. Everything in `/opt/services` (Caddy, deploy-api, health-api, …) is invisible and cannot be restarted through stack-api.

## Endpoints (current)

Signed endpoints: action = payload prefix. Auth is IP + token + signature unless noted.

| Path | Action | Request | Response |
|---|---|---|---|
| `GET /health` | — | — (IP only) | `{"status":"ok"}` |
| `POST /ping` | `stack-ping` | JSON object + `nonce` | `{"ok":true,"principal":…,"scope":…,"request":{…}}` |
| `POST /stacks/list` | `stack-list` | `{nonce}` | `{"stacks":[{name, type, path, sandbox, containers:[{id (12 chars), name, service, image, state, status, ports:[{ip, public, private, type}]}]}]}` |
| `POST /stacks/get` | `stack-get` | `{"name", nonce}` | one stack including `sandbox`; containers also have `started_at`, `restart_count`, `health`. `400` `invalid name`, `404` `stack not found` (also for a sandbox principal asking for a non-sandbox stack) |
| `POST /stacks/restart` | `stack-restart` | `{"name", "service"?, "async"?, nonce}` — `async` only when true | Sync (unchanged without `async`): restarts one service or the whole stack (`db` first); Docker restart `t=10`. `{"name","results":[{service, name, ok}]}`. `async: true` → `{job, state:"queued"}`. `404` `service not found`; `409` `stack busy`; `502` with `results` if any restart failed |
| `POST /stacks/logs` | `stack-logs` | `{name, service, lines?, nonce}` (`lines` 1–1000, default 200) | `{name, service, containers:[{name, state, truncated, lines:[{ts, stream, text}]}]}`. Container stdout/stderr via the Docker logs API (demuxed, timestamps on), max 2 MB per container, each line cut to 4000 chars. Log content is never written to the audit log (detail is only `lines=N`). `404` `service not found` |
| `POST /jobs/get` | `job-get` | `{id, nonce}` (32 lowercase hex) | `{id, action, stack, service, principal, state (queued / running / ok / error), created_at, started_at, finished_at, result, detail}`. `404` `job not found`, also when the job's stack is not visible to the principal |
| `POST /jobs/list` | `job-list` | `{name?, limit?, nonce}` (`limit` 1–100, default 20; omit `name` when null) | `{jobs:[job, …]}` newest first, filtered by principal scope |
| `POST /people/list` | `people-list` | `{nonce}` | `{people:[{name, uid}]}` — no keys |
| `POST /stacks/plan` | `stack-plan` | `{spec, nonce}` | Always 200 for a handled plan. Writes nothing. Rejected: `{ok:false, errors:[string]}`. Planned: `{ok, errors, mode:"create", name, type, path, ssh_port (null without users), ssh_users, compose (YAML, multi-line strings as `\|` blocks), files, caddy, notes}`. `ok` can be false with a rendered result when policy fails |
| `POST /stacks/apply` | `stack-apply` | `{spec, nonce}` | Rejected: `{ok:false, errors:[string]}` (HTTP 200). Accepted: `{ok:true, job, state:"queued", name}`. Heavy job |
| `POST /stacks/fix-permissions` | `stack-fix-permissions` | `{name, nonce}` | `{job, state}`. WordPress stacks under `/opt/apps/wordpress` only; `400` `"only wordpress stacks"` / `"stack folder not available"`. Heavy job |

Both `plan` and `apply` refuse: names not visible to the principal (sandbox), existing stacks (`update is not supported yet`), existing folder, existing Caddy site file. Sandbox principal may only plan/apply names in `STACK_API_SANDBOX_STACKS`.

Docker errors → `502` `"docker error"`.

`ports` includes the bind IP (`ip`, `public`, `private`, `type`) — e.g. `0.0.0.0` and `::` for 2206.

FastAPI docs (`/docs`, `/redoc`, `/openapi.json`) are disabled.

## Spec (WordPress, v1)

Code: `app/stackspec.py` (`validate_spec`, people loader, `render_wordpress`, `render_caddy`, `policy`, `dump_compose`). `pyyaml` is in `requirements.txt`. Empty `php_ini` is a JSON object (`{}`), never `[]`.

| Key | Rule |
|---|---|
| `type` | only `"wordpress"` |
| `name` | exactly one label + `.staging.foxcraft.digital` (`STACK_API_DOMAINS`, wildcard cert covers one level). Label `^[a-z0-9]([a-z0-9-]{0,61}[a-z0-9])?$`, not `atlas` / `api` / `stella` / `edison` / `york` / `www` / `mail`, not `stella-*` |
| `php` | `STACK_API_PHP_VERSIONS` (default `8.5`) |
| `ssh_users` | list of person names from `people/` (max 20, unique; unknown → error). uid + key come from `people/`. Atlas requires at least one |
| `php_ini` | whitelist only; each value validated: `memory_limit`, `upload_max_filesize`, `post_max_size`, `max_execution_time`, `max_input_time`, `max_input_vars`, `display_errors`, `date.timezone`, `session.gc_maxlifetime`, `opcache.enable`, `opcache.memory_consumption`, `opcache.max_accelerated_files`, `opcache.validate_timestamps`, `opcache.revalidate_freq` |

## Policy (allowlist)

Checked on every rendered compose (plan and apply). Top-level keys only `services` / `networks` / `volumes`. Networks only `edge` (external) and `internal`. Named volumes without options. Service keys only `image`, `container_name`, `environment`, `volumes`, `depends_on`, `networks`, `ports`, `restart`, `pull_policy`. Images: `wordpress:php8.x-apache`, `mariadb:11`, `stella/ssh-php:8.x` (`pull_policy: never` only for `stella/`). `container_name` = `<name>` for `wp`, `<name>-<service>` otherwise. `restart: unless-stopped`. Only `wp` on `edge`. Only `ssh` publishes ports, `22100–22199:22`. Bind mounts only `./` paths without `..`. Env values are strings; newlines only in `WORDPRESS_CONFIG_EXTRA`.

## WordPress blueprint

Like hypershape:

| Service | Image | Networks | Notes |
|---|---|---|---|
| `wp` | `wordpress:php<v>-apache` | `edge` + `internal` | `WORDPRESS_CONFIG_EXTRA` with `$$`-escaped HTTPS fix + `WP_HOME` / `WP_SITEURL` |
| `db` | `mariadb:11` | `internal` | named volume `db-data` |
| `ssh` | `stella/ssh-php:<v>` | `internal` | `pull_policy: never`, port `221xx:22`, `./atlas:/etc/atlas:ro`, `home` + `ssh-keys` volumes, `./html:/home/<user>/app` per user |

`./php/zz-atlas.ini` is mounted read-only into `/usr/local/etc/php/conf.d/` in `wp` and `ssh`. No ssh users → no ssh service, no port, no `atlas/` files. Caddy site block has **no `tls` section** (wildcard cert); only `@blocked not remote_ip` from `STACK_API_SITE_ALLOW_IPS` and `reverse_proxy <name>:80`.

**Site allowlist for new stacks (decided 2026-11-08):** `194.126.177.181` (Proton), `78.47.218.190` (york), `2.31.42.13` (Atlas prod).

SSH ports for stack-api managed stacks: **22100–22199**, lowest free; a port freed by a deleted stack can be reused. One `DOCKER-USER` range rule — see [`infrastructure.md`](infrastructure.md) / [`dev-ssh-access.md`](dev-ssh-access.md). Manual ports stay 2201–2207 (next free manual: 2208).

## Jobs

SQLite table `jobs` (`id`, `created_at`, `started_at`, `finished_at`, `principal`, `action`, `stack`, `service`, `state` `queued`/`running`/`ok`/`error`, `result` JSON, `detail`). One in-memory asyncio lock per stack: a second action on a busy stack (sync restart or any job) gets `409` `stack busy`. Heavy-job semaphore `STACK_API_HEAVY_JOBS` (default 1) — `apply` and `fix-permissions`. On startup, queued/running jobs are set to `error` `interrupted by stack-api restart`. Job completion writes a second audit row `job=<id> done …`. Locks and the replay cache require a single uvicorn worker.

## Apply job

Order:

1. Pick SSH port (lowest free in 22100–22199, inside the heavy-job slot so two applies cannot race), render, policy.
2. Files: `mkdir /opt/apps/wordpress/<name>` (fails if it exists), `html/`, `.env` (`chmod 600`, `DB_PASSWORD` + `DB_ROOT_PASSWORD` random hex, never returned/logged), `php/zz-atlas.ini`, `atlas/users.conf` + keys, `docker-compose.yml`.
3. `docker compose up -d --quiet-pull` in the folder (project name from folder).
4. Write `/caddy-sites/<name>.caddy`, `docker exec services-caddy-1 caddy reload --config /etc/caddy/Caddyfile --adapter caddyfile` (a bad site file fails the reload; old config keeps running).
5. Wait up to 180 s for 200/301/302 from `curl http://localhost/` inside the wp container.
6. Fix permissions (below).

On any failure: remove the site file + reload Caddy, `compose down -v --remove-orphans`, delete the folder (only if this job created it). Job result on error: `{step, output, done, rollback}`. Success result: `{name, path, url, ssh_port, ssh_users, steps}`.

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

## Shared SSH image (`stella/ssh-php:<version>`)

Built 2026-11-04. One image per PHP version, used by WordPress stacks' `ssh` service instead of a per-site `Dockerfile.ssh`. First stack: `hypershape`. Pattern: [`wordpress-staging.md`](wordpress-staging.md#ssh-container-shared-image). Updates: [`php-updates.md`](php-updates.md#shared-ssh-image-stellassh-php).

**Source:** `/opt/services/stack-api/images/ssh-php/` — `Dockerfile` (`ARG PHP_VERSION` default `8.5`, `FROM php:${PHP_VERSION}-cli`), `atlas-sync-users`, `entrypoint.sh`.

**Build** (local image only; compose uses `pull_policy: never`):

```bash
docker build --build-arg PHP_VERSION=8.5 -t stella/ssh-php:8.5 /opt/services/stack-api/images/ssh-php
```

**Contents:** `openssh-server`, `git`, `curl`, `wget`, `unzip`, `less`, `procps`, `mariadb-client`, WP-CLI phar at `/usr/local/bin/wp`, group `atlasusers`. Verified: PHP 8.5.12, WP-CLI 2.12.0, MariaDB client 11.8.6.

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
- [x] people list (`people/` baked into the image; spec takes person names)
- [x] `fix-permissions` action (2026-11-08)

### apply (create/change from a spec)

- [x] Spec schema with strict validation (names, PHP versions from a list, whitelisted ini keys, no newlines; paths must resolve under `/opt/apps/<type>/<name>/`) (2026-11-08)
- [x] Policy check on every rendered compose before `up` (refuse `privileged`, `cap_add`, `network_mode: host`, `pid: host`, bind mounts outside the app folder, `docker.sock`, `edge` on `ssh`/`db`/`adminer`, images outside the allowlist) (2026-11-08)
- [x] WordPress blueprint (template of the hypershape compose; `ssh` on `internal`) (2026-11-08)
- [x] SSH users via spec (now via `people/`; uid + key from the baked list, then `atlas-sync-users`) (2026-11-08)
- [ ] People sync to existing stacks (a key change in `people/` does not update running stacks)
- [ ] Outbound SSH key per user (`ssh-keys/generate`, returns only the `.pub`, for GitHub deploy keys)
- [ ] PHP version change as a job (DB dump first, `wp` + `ssh` image together, verify `php -v` / HTTP / fatals / ACL, revert)
- [ ] PHP settings: `PHP_INI_SCAN_DIR` with shared `php-defaults.ini` + per-site `zz-atlas.ini`, whitelisted keys (`memory_limit`, `upload_max_filesize`, `post_max_size`, `max_execution_time`, `max_input_time`, `max_input_vars`, `display_errors`, `date.timezone`, `session.gc_maxlifetime`, selected `opcache.*`); never `auto_prepend_file`, `disable_functions`, `open_basedir`, `extension`; return effective values via `php -i`
- [x] wildcard DNS + wildcard cert for `*.staging.foxcraft.digital` (2026-11-08); apply writes `/caddy-sites/<name>.caddy` and reloads Caddy
- [x] stack-api SSH ports 22100–22199, lowest free, one `DOCKER-USER` range rule (2026-11-08). A port freed by a deleted stack can be reused.
- [ ] Drift detection (hash of rendered files; mismatch blocks apply until "reconcile")
- [ ] Adopt / update existing stacks: hypershape (done by hand) → `dominikliss.foxcraft.digital` → `osgar`, `advoapp` (need .NET blueprint) → `atlas.dev`, `edison.dev` (need Laravel blueprint)

### WordPress extras

- [x] `fix-permissions` action (chmod/SetGID/setfacl after first start or updates) (2026-11-08)
- [ ] Staging guard toggle (mu-plugin: no outgoing mail, noindex, `blog_public=0`)
- [ ] Real cron (`DISABLE_WP_CRON` + scheduled `wp cron event run`)
- [ ] DB dump / import / safe search-replace via WP-CLI
- [ ] Optional basic auth / IP restriction per site in the Caddy block
- [ ] Labels as toggles: `backup.exclude`, `health.enable`
- [ ] Git repos for custom plugins/themes; Edison workspace registration (`/v1/workspaces/ensure`)

### restore / delete / databases

- [ ] `restore`: staging from an Atlas backup as create + restore job (reuse the wp-staging-api code: manifest check, streaming decrypt, collation mapping, search-replace, deactivate mail plugins); also "refresh from newer backup"
- [ ] `delete`: soft delete to `/opt/apps/.trash/<name>-<ts>` incl. Caddy site, volumes kept 7 days, optional final DB dump
- [ ] Adminer on demand per stack (internal network only, one-time auto-login URL, TTL ~30 min, read-only DB user by default on non-sandbox stacks, dump before write access)

### Other stack types

- [ ] Laravel blueprint (`app`, queue, scheduler, `db` mysql 8.4, `ssh` with composer/node; version = build arg; `queue:restart` after changes)
- [ ] .NET blueprint (`-dev` with `dotnet watch` + supervisord `stopasgroup`/`killasgroup`, MSSQL named volume, `ssh` with node/supervisorctl/sqlcmd)

### Hardening / later

- [ ] Optional second confirmation (email/TOTP) for destructive actions on non-sandbox stacks
- [ ] Audit mirror in Atlas
- [ ] WP-CLI Deprecated notice on PHP 8.5 (suppress `E_DEPRECATED` for the `wp` wrapper only?)
