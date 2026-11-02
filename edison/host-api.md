# Edison host API

Headless adapter on the **Edison box**. One FastAPI service (`host-api`) serves the Atlas health snapshot and the web-app `/v1` endpoints. The Laravel web app never runs Cursor or Claude. It only calls this API. The host picks a CLI from `agent` and works in an sshfs workspace.

**Status:** live at `https://api.edison.foxcraft.digital` (2026-11-01). Run options live **2026-11-02**. Workspaces v1 (catalog, on-demand sshfs, repo scan, workspace rule) shipped **2026-11-02**. Health and `/v1/*` are in the same process. Caddy runs in Docker next to it. `edison.dev` on Stella is configured to use it. If `EDISON_HOST_API_URL` is empty, the Laravel app uses its demo driver — the host itself is live.

**Related:** [README.md](README.md) (role), [infrastructure.md](infrastructure.md) (firewall, `/opt/services`), [cursor-agent.md](cursor-agent.md) (container, mounts, `cursor-agent-run`), [run-options.md](run-options.md) (design: modes / models / settings), [app.md](app.md) (Laravel client), Atlas polling (planned): [`../atlas/monitoring.md`](../atlas/monitoring.md).

---

## Decision (2026-11-01)

Health and the `/v1` web-app endpoints live in **one** service (`host-api`) on one subdomain, same as the Atlas host API. Separate auth: SSH signature for `/system/*`, bearer token for `/v1/*`.

Edison uses Docker for Caddy and host-api (Stella pattern), not native systemd like the Atlas server, because Edison is already a Docker host and the API must `docker exec` into `cursor-agent`.

---

## DNS

A record `api.edison` → `178.105.203.54` created via Hetzner Cloud API from Stella (zone `567656`, TTL 300). Token from `/opt/services/caddy/.env` on Stella. Verified with `dig @8.8.8.8`. Do not write the token into docs.

---

## Layout

```
/opt/services/
├── docker-compose.yml    # caddy + host-api, network edison-net (external)
├── caddy/
│   ├── Dockerfile        # identical to Stella: xcaddy + caddy-dns/hetzner/v2
│   ├── Caddyfile
│   └── .env              # HETZNER_API_TOKEN, copied from Stella via scp -3, chmod 600
└── host-api/
    ├── Dockerfile        # python:3.13-slim + docker 29 CLI, WORKDIR /srv, COPY app ./app
    ├── requirements.txt  # fastapi 0.120.*, uvicorn[standard] 0.38.*
    ├── .env              # EDISON_HOST_API_TOKEN only (value not recorded), chmod 600
    ├── allowed_signers   # /system/health signature check
    ├── catalog.json      # host-owned catalog, bind-mounted ro → /etc/host-api/catalog.json
    ├── app/
    │   ├── main.py           # routes
    │   ├── auth.py           # bearer + signed requests
    │   ├── capabilities.py   # modes, model catalog, option mapping
    │   ├── catalog.py        # catalog loader, live mount check, ssh probe, workspace validation
    │   ├── repos.py          # repo scan
    │   ├── mounts.py         # ensure, mounts.json, `remount` CLI
    │   ├── runs.py           # /v1 run lifecycle; workspace = catalog mount or a subdir
    │   └── health.py         # expected mounts = env default ∪ mounts.json
    └── app.bak-2026-11-02*   # rollout backups. TODO (Dominik): delete after a stable week
```

Image `services-host-api`. Deploy: `cd /opt/services && docker compose up -d --build host-api`. Container `host-api` listens on `8080` inside `edison-net`; published only through Caddy as `https://api.edison.foxcraft.digital` (IP allowlist). Python 3.13-slim, FastAPI 0.120, uvicorn 0.38. Docker CLI is copied into the image; the API execs into `cursor-agent` with `docker exec`. Reached from Stella through Caddy: `GET /v1/capabilities` from `/opt/apps/laravel/edison.dev.foxcraft.digital` returned **200** (verified 2026-11-02 with the bearer token).

Named volume `host-api-state` → `/var/lib/host-api/`. Do not write token values into docs.

| File in the volume | Purpose |
|---|---|
| `sessions.json` | Run sessions (workspace + agent per `session_id`) |
| `mounts.json` | Hosts that were ensured; remounted after restart |
| `locks/*.lock` | `flock` files, one per host plus `mounts-file.lock` |

| Container | `container_name` | Ports | Notes |
|-----------|------------------|-------|-------|
| `caddy` | `caddy` | publishes **443 only** | `env_file` `caddy/.env`. Volumes: Caddyfile (ro), `caddy-data`, `caddy-config`. Port 80 is not published. `:80 HTTP/2 skipped` log lines are harmless. |
| `host-api` | `host-api` | none | `env_file` `host-api/.env`. Volumes: `/var/run/docker.sock`, `allowed_signers` (ro), `catalog.json` → `/etc/host-api/catalog.json` (ro), `host-api-state` → `/var/lib/host-api`. |

Both join `edison-net`.

---

## Endpoints

All live.

| Method | Path | Auth | Purpose |
|---|---|---|---|
| GET | `/health` | none | Liveness `{"status":"ok"}` |
| POST | `/system/health` | signed `system-health`, `allowed_signers` | Host snapshot for Atlas |
| GET | `/v1/health` | none | `{"status":"ok","agents":["cursor"]}` (`agents` = `cursor-agent` container running) |
| GET | `/v1/agents` | bearer | `{"current":"cursor","available":[…]}` |
| GET | `/v1/capabilities` | bearer | Modes, model catalog, settings schema for an agent |
| GET | `/v1/workspaces` | bearer | Catalog with live `mounted` / `ssh_ready` flags. Does not mount or scan. |
| POST | `/v1/workspaces/ensure` | bearer | Idempotent sshfs mount for a catalog host |
| GET | `/v1/workspaces/{host}/repos` | bearer | Git repos inside a live mount. Caller must `ensure` first. |
| POST | `/v1/runs` | bearer | Start/resume. Accepts `mode`, `model`, `settings`. SSE (`delta`, `done`, `error`, `: ping` comment every 15 s) or JSON |
| POST | `/v1/runs/abort` | bearer | Kill a running session |

Errors are JSON `{"message": …}`. Validation errors return **400**. Bearer token: `EDISON_HOST_API_TOKEN`. If unset, bearer routes return **503**.

Same SSH-signature scheme as Stella health-api and the Atlas host API (`ssh-keygen -Y sign` / `-Y verify`, namespace `deploy`, principal `ddashboard`, 60 s replay window). See [`../stella-server/health-api.md`](../stella-server/health-api.md) and [`../atlas/host-api.md`](../atlas/host-api.md).

---

## Snapshot fields (`POST /system/health`)

| Key | Contents |
|---|---|
| `host` | `edison` |
| `timestamp` | ISO timestamp |
| `docker_daemon` | `{status, version}` |
| `containers` | `{name: {status, started_at, restart_count}}`. Core list: `caddy`, `host-api`, `cursor-agent`. Missing → `"missing"`. |
| `agent` | `{status, version}` via `cursor-agent --version` inside the container |
| `mounts` | `{path: {status, message?}}`. Expected = `EDISON_EXPECTED_MOUNTS` (default `/mnt/osgar.datahub.foxcraft.digital`; not set in `.env`) ∪ every host in `mounts.json`, no duplicates. `health.py` reads `mounts.json` directly (no import of `mounts.py`, to avoid a cycle). Checks mountpoint + `ls` with 8 s timeout. Verified 2026-11-02: osgar and dominikliss report `up`. **UNVERIFIED:** whether the Atlas Edison card (`/monitoring/edison`) shows the extra mount rows. **TODO (Dominik):** check. |
| `certs` | `{domain: {expires, days_left, status}}` via TLS to `caddy:443` with SNI |
| `disk` | `{total_gb, free_gb, used_pct}` (container root = host disk) |
| `cpu_ram` | `{cpu_threads, load_1m, load_5m, load_15m, ram_total_gb, ram_available_gb, ram_used_pct}` from `/proc` |

---

## Caddyfile

Certificate for `api.edison.foxcraft.digital` obtained from Let's Encrypt production on the first attempt (DNS-01, ~100 s, `propagation_delay 90s` + `propagation_timeout 5m`, `dir` pinned). New ACME account registered for `projects@foxcraft.digital`.

```caddyfile
api.edison.foxcraft.digital {
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
        not remote_ip 2.31.42.13 95.217.144.93 194.126.177.181
    }
    respond @blocked 403
    @system_blocked {
        path /system/*
        not remote_ip 2.31.42.13 95.217.144.93
    }
    respond @system_blocked 403
    @v1_blocked {
        path /v1/*
        not remote_ip 95.217.144.93 194.126.177.181
    }
    respond @v1_blocked 403
    reverse_proxy host-api:8080 {
        flush_interval -1
    }
}
```

| Matcher | When | Result |
|---------|------|--------|
| `@blocked` | not Atlas prod / Stella / MacBook | 403 |
| `@system_blocked` | path `/system/*` and not Atlas prod / Stella | 403 |
| `@v1_blocked` | path `/v1/*` and not Stella / MacBook | 403 |

`flush_interval -1` disables SSE buffering.

---

## allowed_signers

Two `ddashboard` lines. Same keys as Stella health-api.

| Comment | Who |
|---------|-----|
| `foxcraft-digital@dedi6724` | atlas-prod |
| (none) | atlas-dev |

---

## Role

```
Browser  →  Edison web app (Laravel, Stella)
                POST /api/chats/{id}/messages   (SSE to the browser)
                     ↓
                POST {EDISON_HOST_API_URL}/v1/runs
                     ↓
            Edison host API  (this service)
                     ↓
            cursor-agent-run | claude  in /mnt/<host>
```

The web app is agent-agnostic. Switching Cursor → Claude is `EDISON_AGENT=claude` plus a host implementation. Do not add vendor CLIs to the Laravel repo.

---

## What the web app calls

| When | Method | Path | Accept |
|---|---|---|---|
| User sends a message (UI) | `POST` | `/v1/runs` | `text/event-stream` |
| JSON fallback / tests | `POST` | `/v1/runs` | `application/json` |

Connect timeout: 5 s. Read timeout: `EDISON_AGENT_TIMEOUT` (**900** on `edison.dev`, raised from 180 to match the host deadline). If `EDISON_HOST_API_TOKEN` is set, the app sends `Authorization: Bearer <token>`.

The app on `edison.dev` is pointed at this API. The Laravel workspaces contract (catalog → ensure → repos → run on a leaf) is **not built yet** — see [Web app contract](#web-app-contract-not-built-yet-in-laravel). **UNVERIFIED:** end-to-end chat in the `edison.dev` UI (including resume via the app).

---

## `GET /v1/health`

Liveness for Docker / Caddy / uptime. No auth.

**200**

```json
{ "status": "ok", "agents": ["cursor"] }
```

`agents` is the list this process can actually exec (`cursor-agent` container running). Omit Claude until it is installed.

---

## `GET /v1/agents`

Bearer required.

```json
{ "current": "cursor", "available": ["cursor"] }
```

---

## `GET /v1/capabilities`

Bearer required. Query `agent` (optional, default `cursor`). `claude` → **400** `Agent claude is not installed.`; anything else unknown → **400** `Unknown agent {agent}.`

**200** (live 2026-11-02: 3 modes, **270** models, `auto` first and the only `default: true`)

```json
{
  "agent": "cursor",
  "current": true,
  "modes": [
    { "id": "agent", "label": "Agent", "description": "Full tools. May edit the workspace.", "writes": true },
    { "id": "plan",  "label": "Plan",  "description": "Design an approach. Should not edit source.", "writes": false },
    { "id": "ask",   "label": "Ask",   "description": "Read-only questions. Should not edit files.", "writes": false }
  ],
  "default_mode": "agent",
  "models": [
    { "id": "auto", "label": "Auto", "default": true },
    { "id": "gpt-5.3-codex-low", "label": "Codex 5.3 Low" }
  ],
  "settings": []
}
```

`writes` is advisory for the UI. `--mode=plan|ask` is **not** a security boundary.

**Model catalog:** `docker exec cursor-agent cursor-agent models`. Line format `<id> - <label>`, plus an `Available models` header and a `Tip:` footer. Parser keeps only `^([A-Za-z0-9][A-Za-z0-9._-]*) - (.+)$`, strips ANSI and zero-width characters (some Grok 4.7 labels contain them), collapses double spaces, strips the `(current, default)` marker. `auto` is always first.

Cache: 60 s in process memory. On failure: keep the last good catalog for 15 s more; if there never was one, return only `auto`. Never 502.

The catalog is large (~270). The Laravel UI needs search or grouping, not a plain dropdown. See [run-options.md](run-options.md).

---

## Catalog

Host-owned, not Laravel-owned. File: `/opt/services/host-api/catalog.json`, bind-mounted read-only to `/etc/host-api/catalog.json`. One row per Stella project. The API never grows the catalog. Adding a host = edit `catalog.json` + Stella SSH container, `edison` user/key and `DOCKER-USER` rule. The file is re-read on every request (no restart after an edit).

| host | kind | ssh_port | ssh_user | remote_path |
|---|---|---|---|---|
| osgar.datahub.foxcraft.digital | dotnet | 2201 | edison | /home/edison/app |
| advoapp.finditoo.foxcraft.digital | dotnet | 2202 | edison | /home/edison/app |
| dominikliss.foxcraft.digital | wordpress | 2204 | edison | /home/edison/app |
| atlas.dev.foxcraft.digital | laravel | 2205 | edison | /home/edison/app |
| hypershape.foxcraft.digital | wordpress | 2206 | edison | /home/edison/app |
| edison.dev.foxcraft.digital | laravel | 2207 | edison | /home/edison/app |

Validation on load: host must be a lowercase FQDN (no IP, no short name, no `..`), `kind` ∈ `dotnet|laravel|wordpress`, port 1–65535, `ssh_user` a plain Unix name, `remote_path` absolute without `..`. Duplicate host or invalid row → **500** `Catalog is invalid: …`.

Derived, never stored: `mount = /mnt/{host}`.

Fallback: if `catalog.json` is missing, `EDISON_WORKSPACES` (comma-separated `/mnt/<host>`) is used with unknown kind/port. It is not set today. Plan: drop the fallback after one release.

The host lists repos **found in a mount** (scan only). It does not catalog GitHub repos. Still out of scope: chat CRUD, ddashboard auth.

---

## `GET /v1/workspaces`

Bearer required. Lists the catalog with live flags. Does not mount or scan.

```json
{
  "workspaces": [
    {
      "host": "osgar.datahub.foxcraft.digital",
      "kind": "dotnet",
      "ssh_port": 2201,
      "mount": "/mnt/osgar.datahub.foxcraft.digital",
      "path": "/mnt/osgar.datahub.foxcraft.digital",
      "mounted": true,
      "ssh_ready": true
    }
  ]
}
```

- `path` is kept equal to `mount` for the current Laravel client (it reads `{path, mounted}`). Remove once Laravel uses `mount`.
- `mounted`: `mountpoint -q` and `ls` within 8 s, inside `cursor-agent`.
- `ssh_ready`: TCP connect to `{host}:{ssh_port}` from inside `cursor-agent`, 3 s (`EDISON_SSH_PROBE_TIMEOUT`). `true` means the port answers, not that `edison` auth works.

Verified 2026-11-02 from Stella: 6 rows, osgar `mounted: true`, all `ssh_ready: true`.

---

## `POST /v1/workspaces/ensure`

Bearer required. Body `{ "host": "<catalog FQDN>" }`. Idempotent. Mounts the host inside `cursor-agent`. Catalog hosts only.

**200**

```json
{ "host": "dominikliss.foxcraft.digital", "kind": "wordpress", "mount": "/mnt/dominikliss.foxcraft.digital", "mounted": true, "already_mounted": false }
```

| Status | message |
|---|---|
| 400 | `Unknown host '{host}'.` |
| 400 | `host: Field required` (validation handler) |
| 401 | bad/missing bearer |
| 409 | `Another mount operation is still running.` (lock not free within `EDISON_MOUNT_TIMEOUT`) |
| 422 | `SSH port {n} on {host} is not reachable.` |
| 422 | `Failed to mount {host}: {sshfs stderr, max 300 chars}` |
| 503 | `cursor-agent is not running.` |

Procedure:

1. Resolve the catalog row (unknown → 400). Check `cursor-agent` is running (else 503).
2. Take the per-host file lock (`/var/lib/host-api/locks/{host}.lock`, `flock`). Works across processes, so the API and the `remount` CLI cannot double-mount.
3. Live check passes → persist host in `mounts.json`, return `already_mounted: true`.
4. Otherwise clean up first: if `/mnt/{host}` is a mountpoint (dead mount, e.g. "Transport endpoint is not connected") → `umount -l`; then `rmdir` if empty.
5. TCP probe; fail → 422.
6. `mkdir -p /mnt/{host}` and `sshfs` with everything on the command line (the API does not depend on `/root/.ssh/config`):

```
sshfs -o reconnect,ServerAliveInterval=15,ServerAliveCountMax=3,IdentityFile=/root/.ssh/id_ed25519,IdentitiesOnly=yes,StrictHostKeyChecking=accept-new,UserKnownHostsFile=/root/.ssh/known_hosts,UpdateHostKeys=no,BatchMode=yes,ConnectTimeout=8 \
  -p {ssh_port} {ssh_user}@{host}:{remote_path} /mnt/{host}
```

   - `accept-new` is needed for hosts never used before. Never `StrictHostKeyChecking=no`.
   - `UpdateHostKeys=no` because the single bind-mounted `known_hosts` cannot be rewritten (`Invalid cross-device link`).
   - `BatchMode=yes` so a host that offers password auth fails instead of waiting for a prompt.
7. Live check again. Fail → `umount -l` + `rmdir`, then 422. No empty directory is left behind.
8. Persist and return 200.

`sshfs` gets `EDISON_MOUNT_TIMEOUT` (default 20 s). Lock wait has the same limit.

For a rejected key, sshfs reports `read: Connection reset by peer`, not ssh's `Permission denied`. The 422 is correct; the text is less specific.

There is no unmount API in v1. To drop a host: edit `mounts.json`, then `umount -l` inside `cursor-agent`.

---

## `GET /v1/workspaces/{host}/repos`

Bearer required. Does not mount. Caller must `ensure` first.

| Status | message |
|---|---|
| 400 | `Unknown host '{host}'.` |
| 404 | `Workspace {mount} is not mounted.` |
| 502 | `Repo scan failed on {host}.` (scan command failed over sshfs) |

`dotnet` / `laravel`: always exactly one repo, the mount root. `wp-content` is not looked at. `name` = host without `.foxcraft.digital`.

```json
{ "host": "osgar.datahub.foxcraft.digital", "kind": "dotnet", "mounted": true,
  "repos": [ { "slug": "app", "name": "osgar.datahub", "kind": "app", "workspace": "/mnt/osgar.datahub.foxcraft.digital" } ] }
```

`wordpress`: scans only `{mount}/wp-content/themes/*/` and `{mount}/wp-content/plugins/*/`. A directory is included when `{dir}/.git` exists (directory or gitfile). No recursion, no `mu-plugins`, no single-file plugins. Symlinked theme/plugin folders are skipped (prevents symlink escape). Names must match `[A-Za-z0-9._-]+`. Sorted themes first, then plugins, then name. Missing `wp-content` → `repos: []` with 200.

**Deviation from the original spec:** `slug` is `theme/<name>` or `plugin/<name>` (not just the basename), so a theme and a plugin with the same folder name cannot collide. `name` is the folder name.

Live result 2026-11-02:

```json
{ "host": "dominikliss.foxcraft.digital", "kind": "wordpress", "mounted": true,
  "repos": [
    { "slug": "theme/dominikliss", "name": "dominikliss", "kind": "theme", "workspace": "/mnt/dominikliss.foxcraft.digital/wp-content/themes/dominikliss" },
    { "slug": "plugin/atlas-connect", "name": "atlas-connect", "kind": "plugin", "workspace": "/mnt/dominikliss.foxcraft.digital/wp-content/plugins/atlas-connect" }
  ] }
```

Known git checkouts on WordPress hosts (checked 2026-11-02 over SSH):

| Host | Repos (`.git` present) |
|---|---|
| dominikliss.foxcraft.digital | `themes/dominikliss`, `plugins/atlas-connect` |
| hypershape.foxcraft.digital | `plugins/hypershape-forms` |

---

## `POST /v1/runs`

Start or resume a run.

**Headers**

| Header | Required | Notes |
|---|---|---|
| `Content-Type` | yes | `application/json` |
| `Accept` | yes | `text/event-stream` when `stream` is true; `application/json` otherwise |
| `Authorization` | yes | `Bearer <EDISON_HOST_API_TOKEN>` |

**Body**

```json
{
  "prompt": "Fix the ingest timeout",
  "workspace": "/mnt/osgar.datahub.foxcraft.digital",
  "resume": null,
  "stream": true,
  "agent": "cursor",
  "mode": "ask",
  "model": "auto",
  "settings": {}
}
```

| Field | Type | Required | Meaning |
|---|---|---|---|
| `prompt` | string | yes | User text. 1–20000 characters (app already validates). |
| `workspace` | string | yes | Absolute path **inside `cursor-agent`**. Must be a catalog mount or a subdirectory of one. The host does **not** require `.git` and does **not** auto-ensure on run. |
| `resume` | string \| null | no | Previous `session_id` for this chat. `null` = new session. Prefix is `cursor_<uuid>`. |
| `stream` | boolean | no | Default `false`. UI always sends `true`. |
| `agent` | string | yes | `cursor` now. `claude` later. Unknown value → **400**. |
| `mode` | string | no | Default `agent`. `agent`, `plan`, or `ask`. Per run; not stored in `sessions.json`. Resume with a different mode is allowed. |
| `model` | string | no | Default `auto`. `auto` / `""` / `null` = Auto. Otherwise must be a catalog id. If the catalog could not be loaded, any id matching `^[A-Za-z0-9][A-Za-z0-9._-]{0,63}$` is passed through and Cursor decides. |
| `settings` | object | no | Default `{}`. Cursor v1 supports no keys. Unknown key → **400**. |

Omitting `mode`, `model`, and `settings` gives exactly the pre-change argv.

### Workspace rule (2026-11-02)

A workspace is valid when all of these hold:

1. Absolute, no `..` segment, no `//`, no NUL, not `/`, `/root` or `/mnt`.
2. After `posixpath.normpath` it equals `/mnt/{catalog host}` or is below one catalog mount.
3. That mount is live → else **404** `Workspace {mount} is not mounted.`
4. Inside `cursor-agent`, `realpath -e` resolves to an existing directory that is still under that mount (no symlink escape) → else **400** `Workspace is not a directory under an allowed mount.`

Steps 1–2 run before the resume checks; steps 3–4 run after them (where the old `is_mounted` check was). The normalized path is what gets stored in `sessions.json`, so resume compares normalized paths.

The `GUARD` text (no `git checkout/switch/stash/reset/commit`) is unchanged.

Validation errors (all **400**):

- `Unknown mode '{mode}' for agent cursor.`
- `Unknown model '{model}' for agent cursor.`
- `Unknown setting '{key}' for agent cursor.`
- `settings must be an object.`

Parameterized model ids like `claude-opus-4-8[effort=high]` are rejected (**400**) in v1. `settings.effort` may map to them later.

### Streaming response (`stream: true`)

**200** + `Content-Type: text/event-stream` + `Cache-Control: no-cache` + `X-Accel-Buffering: no`.

```
event: delta
data: {"text":"I'll look through "}

event: delta
data: {"text":"osgar.datahub.\n"}

: ping

event: done
data: {"output":"I'll look through osgar.datahub.\n…","session_id":"cursor_<uuid>"}
```

Rules:

- Flush every `delta`. Do not buffer the whole run.
- `text` is an increment, not the full reply so far.
- `done.output` is the full assistant text. The app concatenates deltas; it uses `output` only if it received no deltas.
- `session_id` is required on `done`. The app stores it on the chat and sends it back as `resume`. Prefix is `cursor_`.
- Comment lines starting with `:` (`: ping` every 15 s) keep the connection alive. The app should ignore them.
- Optional flags on `done`: `"timeout": true` (host deadline), `"aborted": true`.
- One blank line after each event (`\n\n`).
- If the client drops, stop the CLI (SIGTERM, then SIGKILL).

On a stream that has already started, emit then close:

```
event: done
data: {"output":"<text so far>","session_id":"cursor_<uuid>","timeout":true}
```

or:

```
event: error
data: {"message":"cursor-agent exited 1"}
```

### JSON response (`stream: false`)

**200** + `Content-Type: application/json`

```json
{
  "output": "I'll look through osgar.datahub.\n…",
  "session_id": "cursor_<uuid>"
}
```

Host deadline exceeded in JSON mode → **504**.

---

## `POST /v1/runs/abort`

Stop an in-flight run.

**Body**

```json
{ "session_id": "cursor_<uuid>", "agent": "cursor" }
```

**200**

```json
{ "status": "aborted", "session_id": "cursor_<uuid>" }
```

**404** if the session is unknown or already finished.

Known limitation: on the first turn of a chat the app only learns `session_id` with `done`, so the first turn cannot be aborted. **TODO:** emit an early `session` SSE event.

---

## Run implementation (`runs.py`)

Always:

```
docker exec -w <workspace> cursor-agent setsid -w sh -c '<write pid file>; exec cursor-agent-run "$@"' <run_id> -p --trust … 
```

Guard text is prepended to every prompt: unattended run, live staging tree, no `git checkout` / `switch` / `stash` / `reset` / `commit`. PID file: `/tmp/host-api-runs/<run_id>.pid` inside `cursor-agent`. `setsid` gives the run its own process group. Abort/kill uses `/usr/bin/kill -TERM -- -<pgid>`, waits up to 5 s, then KILL. Gotcha found 2026-11-01: dash's builtin `kill` rejects `--` (`Illegal number: -`) and the first version left the Cursor process running. Fixed by calling `/usr/bin/kill` explicitly. Host deadline `EDISON_RUN_TIMEOUT` default **900 s** → `done` with partial output + `"timeout": true`, or **504** in JSON mode. `CURSOR_API_KEY` is still not set. Interactive login is used. **TODO (Dominik):** API key for unattended runs.

### Cursor mapping (verified 2026-11-02)

CLI version on Edison: **`2026.10.22-27dcf5f`**. There is no `--mode=agent` in the CLI. Agent mode = no `--mode` flag.

| `mode` | Extra argv | `--force` |
|---|---|---|
| `agent` (default) | none | **yes** |
| `plan` | `--mode=plan --sandbox enabled` | **no** |
| `ask` | `--mode=ask --sandbox enabled` | **no** |

Then `--model <id>` when the model is not Auto, or `--model $EDISON_CURSOR_MODEL` when Auto and that env is set (ops pin wins over Auto; an explicit client model wins over the pin). Then `--resume <uuid>`, then the prompt.

`--sandbox enabled` is dropped when `EDISON_SANDBOX_READONLY` is `0` / `false` / `no`. `--mode=plan|ask` is not a security boundary. `writes: false` is advisory. Guard text still applies. A separate worktree is still **TODO (Dominik)**.

### Cursor `stream-json` (verified 2026-11-01)

`cursor-agent` **2026.10.22-27dcf5f**.

| Vendor event | Mapping |
|---|---|
| `system/init` | `session_id` UUID, model `"Auto"`, `apiKeySource` `"login"` |
| `user` | ignored |
| `thinking` deltas | ignored |
| `assistant` | text in `message.content[].text` → SSE `delta` |
| `result` | `result`, `is_error`, subtype `success` → SSE `done` |

`session_id` returned to the app as `cursor_<uuid>`.

---

## Sessions

`sessions.json` (named volume `host-api-state`) stores only `agent`, `workspace`, `created_at`, `last_used_at`. No prompts. `mode` / `model` / `settings` are per run and are not stored here.

| Concept | Owner | Id |
|---|---|---|
| Chat in the web app | Laravel DB | integer `chats.id` |
| Vendor CLI session | Host / CLI | string `cursor_<uuid>` |

Resume rules:

| Case | Status |
|---|---|
| Id does not match `cursor_<uuid>` (e.g. literal `<SESSION_ID>`) | **400** `Invalid resume id.` |
| Unknown id | **400** |
| Other workspace or agent | **400** |
| Same session already running | **409** |
| More than `EDISON_MAX_RUNS` (default 3) | **429** |
| Workspace fails the path/catalog rule | **400** |
| Workspace mount not live | **404** |

---

## Errors

Always JSON `{ "message": "…" }` except on an open SSE stream (then send a last event and close).

| Status | When |
|---|---|
| 400 | Missing/invalid field, unknown `agent` / `mode` / `model` / settings key / host, workspace not under a catalog mount, invalid / unknown resume id |
| 401 | Missing or bad bearer token |
| 403 | Caddy IP allowlist (`@blocked` / `@system_blocked` / `@v1_blocked`) |
| 404 | Workspace not mounted; abort of an unknown or finished session |
| 409 | Session already running; another mount operation still running |
| 422 | SSH port not reachable; sshfs failed |
| 429 | More than `EDISON_MAX_RUNS` concurrent runs |
| 502 | CLI crashed or produced no usable output; repo scan failed over sshfs |
| 503 | `EDISON_HOST_API_TOKEN` unset (bearer routes); `cursor-agent` not running (`ensure`) |
| 504 | CLI exceeded the host’s own deadline (JSON mode) |

---

## Auth and network

- Bind on `edison-net`. Caddy is the only published ingress (443).
- `/system/*` is SSH-signed (`system-health`) plus Caddy IP allowlist (Atlas prod + Stella).
- `/v1/*` requires `Authorization: Bearer <EDISON_HOST_API_TOKEN>` plus Caddy IP allowlist (Stella + MacBook).
- This token is **not** a ddashboard user token. Users never call the host API.
- Do not expose the API to the public internet.

---

## Agent backends

`agent` selects the process. The HTTP contract stays the same.

### `cursor` (now)

Working directory: `workspace`. Mapping: [Cursor mapping](#cursor-mapping-verified-2026-11-02). Notes from [cursor-agent.md](cursor-agent.md):

- Use **`cursor-agent-run`** (umask `002`), never raw `cursor-agent`, if the run may write files.
- `--trust` is required in a new directory.
- `--force` is used in **agent** mode only (print mode writes on the live osgar tree). Plan/ask omit `--force` and add `--sandbox enabled`. A separate worktree is still an open TODO.
- `--resume <uuid>` continues the chat (the `cursor_` prefix is stripped). Omit on first turn. Mode may change on resume.
- `--model` is omitted for Auto unless `EDISON_CURSOR_MODEL` is set. The client can pick any catalog id.
- Do not run `git checkout`, `switch`, `stash`, `reset`, or `commit` on the live staging tree. The guard text is the current constraint.

### `claude` (later)

Same `POST /v1/runs` body. Host execs Claude Code in `workspace` and maps its stream onto the same `delta` / `done` events. Until then, `agent: "claude"` → **400** `{ "message": "Agent claude is not installed." }`.

---

## Persistence and reboot

`mounts.json` (`/var/lib/host-api/mounts.json`) is written atomically (tmp + rename) under `locks/mounts-file.lock`:

```json
{ "hosts": ["osgar.datahub.foxcraft.digital", "dominikliss.foxcraft.digital"], "updated_at": "2026-11-02T21:56:16Z" }
```

`ensure` upserts. There is no unmount API in v1.

CLI (not an HTTP route): `docker exec host-api python -m app.mounts remount`. Runs `ensure` for every host in `mounts.json`, prints one line per host (`mounted`, `already mounted`, or `<status> <message>`), never stops on one failing host.

`/root/cursor-agent/run.sh` (systemd `cursor-agent.service` at boot) recreates the container, waits for `host-api`, then calls `remount`. The old hard-coded `sshfs … osgar` lines are gone. osgar is remounted because it is in `mounts.json`. See [cursor-agent.md](cursor-agent.md).

If `cursor-agent` restarts on its own (Docker restart policy, not via `run.sh`), mounts are lost until something calls `ensure` or `remount`. `GET /v1/workspaces` and health show `mounted: false` in that case.

---

## Environment (host-api container)

| Variable | Default | Meaning |
|---|---|---|
| `EDISON_HOST_API_TOKEN` | — | Bearer token (required). Unset → bearer routes **503**. In `.env`. |
| `EDISON_CATALOG_PATH` | `/etc/host-api/catalog.json` | Catalog inside the container (host file `/opt/services/host-api/catalog.json`) |
| `EDISON_WORKSPACES` | unset | Fallback only if the catalog file is missing. Not set today. Drop after one release. |
| `EDISON_MOUNT_TIMEOUT` | `20` | Seconds for sshfs and lock wait |
| `EDISON_SSH_PROBE_TIMEOUT` | `3` | TCP probe |
| `EDISON_EXPECTED_MOUNTS` | osgar mount | Health base list, unioned with `mounts.json`. Not set in `.env`. |
| `EDISON_STATE_DIR` | `/var/lib/host-api` | sessions, mounts, locks |
| `EDISON_AGENT_CONTAINER` | `cursor-agent` | Agent container |
| `EDISON_CURSOR_MODEL` | empty | Ops pin used when the client sends Auto |
| `EDISON_SANDBOX_READONLY` | `1` | `0`/`false`/`no` drops `--sandbox enabled` from plan/ask runs |
| `EDISON_MODELS_TIMEOUT` | `8` | Seconds for `cursor-agent models` |
| `EDISON_MAX_RUNS` | `3` | Parallel runs; more → **429** |
| `EDISON_RUN_TIMEOUT` | `900` | Host deadline per run, seconds (→ **504** or partial `done` with `timeout: true`) |
| `EDISON_SSE_PING` | `15` | SSE keep-alive comment interval, seconds |

`docker-compose.yml` `host-api` volumes:

```yaml
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ./host-api/allowed_signers:/etc/host-api/allowed_signers:ro
      - ./host-api/catalog.json:/etc/host-api/catalog.json:ro
      - host-api-state:/var/lib/host-api
```

Do not write token values into docs.

---

## Tests (2026-11-01)

| From | Call | Result |
|------|------|--------|
| MacBook | `GET /health` | `{"status":"ok"}` |
| MacBook | `POST /system/health` | **403** (IP not allowed). Correct. |
| MacBook | `GET /v1/health` | ok |
| MacBook | `GET /v1/workspaces` without token | **401** `message` |
| MacBook | `GET /v1/workspaces` with token | only osgar mounted |
| Stella, atlas-dev key (`/opt/apps/laravel/atlas.dev.foxcraft.digital/deploy-key/id_ed25519`) | signed `POST /system/health` | Full snapshot. |
| Streaming run | `POST /v1/runs` | `delta` + `done` with `cursor_<uuid>` |
| Resume, JSON mode | `POST /v1/runs` | same `session_id`; remembered the earlier reply |
| Literal `<SESSION_ID>` as resume | `POST /v1/runs` | **400** `Invalid resume id.` |
| Abort (after `/usr/bin/kill` fix) | `POST /v1/runs/abort` | Cursor process gone |
| `edison.dev` app container on Stella | `GET /v1/health` | ok (firewall + Caddy path) |

Snapshot values at **2026-11-01 02:56 UTC:** Docker 29.9.0, all 3 containers up, cursor-agent `2026.10.01-e373342`, osgar mount up, cert valid until Jan 30 2027 (89 days), disk 9.9 %, load 0.52, RAM 10.1 %. CLI later auto-updated to **`2026.10.22-27dcf5f`**.

### Run options (2026-11-02)

| Test | Result |
|---|---|
| Ask mode with `--sandbox enabled`, prompt asked to create `sandbox-test.txt` | Listed folders, refused to write; file did not exist afterwards. Reads on the sshfs mount work with sandbox enabled. |
| `GET /v1/capabilities` inside the container | `cursor ['agent','plan','ask']`, 270 models, `auto` first |
| `POST /v1/runs` `mode=ask` `model=auto` on osgar | argv `… -p --trust --output-format stream-json --mode=ask --sandbox enabled …`, no `--force`; returned output + `session_id` |
| Resume that session with `mode=agent` `model=composer-2.5` | argv `… -p --trust --force --output-format stream-json --model composer-2.5 --resume <uuid> …`; same `session_id` returned |
| `GET /v1/capabilities` from Stella through Caddy | **200** |
| No token | **401** |
| `?agent=claude` | **400** |
| No `mode`/`model` | identical argv to before |
| `mode=plan` | `--mode=plan` |
| `mode=debug` | **400** |
| `model=not-a-model` | **400** |
| Unknown setting | **400** |
| `settings` not an object | **400** |

Known cosmetic issue (not from this change): in JSON mode, `output` comes from Cursor's `result` field, which joins assistant messages without a separator (e.g. `codebase.A user action`). Streamed deltas are fine.

### Workspaces v1 (2026-11-02, from Stella with the live bearer)

| # | Call | Result |
|---|---|---|
| 1–2 | `GET /v1/workspaces` | 6 rows, osgar `mounted: true`, all `ssh_ready: true` |
| 3 | ensure `not-a-box.foxcraft.digital` | 400 Unknown host |
| 4 | ensure osgar | 200 `already_mounted: true` |
| 5 | repos osgar | one `app` repo |
| 6 | ensure dominikliss | 200 `already_mounted: false` |
| 7 | ensure dominikliss again | 200 `already_mounted: true` |
| 8 | repos dominikliss | `theme/dominikliss`, `plugin/atlas-connect` |
| 9 | repos dominikliss before mount | 404 |
| 10 | run in `/mnt/dominikliss…/wp-content/themes/dominikliss` | 200 `OK` + session |
| — | run in `/mnt/osgar…/Models` (subdir) | 200 `OK` + session |
| 11 | run `/mnt/dominikliss…/../osgar…`, `/mnt`, `/mnt/unknown…`, missing subpath | 400 |
| 12 | run in unmounted hypershape | 404 |
| 13 | ensure advoapp (key rejected) | 422 `Failed to mount … Connection reset by peer`, no leftover dir under `/mnt` |
| 14 | `systemctl restart cursor-agent` | both mounts back from `mounts.json`, `git status` works |

Not tested: full reboot of the Edison box (only the service restart).

**UNVERIFIED:**

- Signed `POST /system/health` from Atlas prod (`2.31.42.13`) with the atlas-prod key.
- End-to-end chat in the `edison.dev` UI, including resume via the app.
- Whether the Atlas Edison card (`/monitoring/edison`) shows extra mounts from `mounts.json`. **TODO (Dominik):** check.

---

## Limitations

| Item | Notes |
|------|-------|
| First-turn abort | App only learns `session_id` on `done`. **TODO:** emit an early `session` SSE event. |
| Live tree | Agent-mode runs use `--force` on the live osgar staging tree. Guard text forbids git checkout / switch / stash / reset / commit. Separate worktree still **TODO (Dominik)**. |
| Mounts | On demand via `ensure` (catalog hosts only). Currently mounted: osgar, dominikliss. advoapp returns **422** (key rejected). No unmount API in v1. |
| Default model | `auto`. Catalog is served by `GET /v1/capabilities` (~270). Ops pin via `EDISON_CURSOR_MODEL` when the client sends Auto. |
| `CURSOR_API_KEY` | Not set. Interactive login is used. |

---

## Gotchas

- After `rm` + `nano` of the Caddyfile use `docker compose up -d --force-recreate caddy` (new inode; `restart` keeps the old file). Same class as Stella Issue 4.
- First build of the Caddy image took several minutes (`go build` prints nothing while compiling on the CX33).
- dash builtin `kill` rejects `--`. Abort must call `/usr/bin/kill`.

---

## Example: first streamed turn

Request:

```http
POST /v1/runs HTTP/1.1
Host: api.edison.foxcraft.digital
Authorization: Bearer $EDISON_HOST_API_TOKEN
Accept: text/event-stream
Content-Type: application/json

{
  "prompt": "What starts the ingest pipeline?",
  "workspace": "/mnt/osgar.datahub.foxcraft.digital",
  "resume": null,
  "stream": true,
  "agent": "cursor"
}
```

Response:

```
HTTP/1.1 200 OK
Content-Type: text/event-stream
Cache-Control: no-cache
X-Accel-Buffering: no

event: delta
data: {"text":"Ingest starts in `Program.cs`.\n"}

event: done
data: {"output":"Ingest starts in `Program.cs`.\n","session_id":"cursor_<uuid>"}
```

Follow-up from the same chat sends `"resume": "cursor_<uuid>"`.

---

## Example: Ask run, then resume as agent (2026-11-02)

`POST /v1/runs` on osgar, `mode=ask`, `model=auto`. Host argv (verified):

```
-p --trust --output-format stream-json --mode=ask --sandbox enabled
```

No `--force`. Returned output + `session_id`. A separate Ask run that asked to create `sandbox-test.txt` listed folders and refused to write; the file did not exist afterwards.

Resume that session with `mode=agent` and `model=composer-2.5`:

```json
{
  "prompt": "<follow-up>",
  "workspace": "/mnt/osgar.datahub.foxcraft.digital",
  "resume": "cursor_<uuid>",
  "stream": false,
  "agent": "cursor",
  "mode": "agent",
  "model": "composer-2.5"
}
```

Host argv (verified): `-p --trust --force --output-format stream-json --model composer-2.5 --resume <uuid>`. Same `session_id` returned.

---

## Web app contract (not built yet in Laravel)

1. `GET /v1/workspaces` → candidate list for "Add project". Drop the static `mounted` flags in `config/edison.php`. Disable Add when `ssh_ready` is false.
2. On Add / first open → `POST /v1/workspaces/ensure`.
3. Then `GET /v1/workspaces/{host}/repos`.
4. 0 repos → empty state; 1 → open chat directly; N → second-level picker (themes, then plugins).
5. `POST /v1/runs` with `workspace` = the leaf `repos[].workspace`.
6. Chats are keyed by `host + slug`, not by host alone. A chat stays on its workspace (resume across workspaces is rejected).

Then drop `path` from the host response once Laravel reads `mount`.

---

## Out of scope

- ddashboard login
- Chat history (Laravel `chats` / `chat_messages`)
- GitHub repo catalog (the host only scans `.git` dirs already on a mount)
- Running builds/tests inside the Edison container (no SDK there — SSH to Stella if needed)
- Writing Atlas or Stella features into this API
