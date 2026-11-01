# Edison host API

Headless adapter on the **Edison box**. The Laravel web app never runs Cursor or Claude. It only calls this API. The host picks a CLI from `agent` and works in an sshfs workspace.

**Status:** health part live at `https://api.edison.foxcraft.digital` (2026-11-01). `/v1` endpoints still not built. Until then the Edison app uses `EDISON_HOST_API_URL` (empty = demo driver).

**Related:** [README.md](README.md) (role), [infrastructure.md](infrastructure.md) (firewall, `/opt/services`), [cursor-agent.md](cursor-agent.md) (container, mounts, `cursor-agent-run`), [app.md](app.md) (Laravel client), Atlas polling (planned): [`../atlas/monitoring.md`](../atlas/monitoring.md).

---

## Decision (2026-11-01)

Health and the `/v1` web-app endpoints live in **one** service (`host-api`) on one subdomain, same as the Atlas host API. Separate auth: SSH signature for `/system/*`, bearer token for `/v1/*`.

Edison uses Docker for Caddy and host-api (Stella pattern), not native systemd like the Atlas server, because Edison is already a Docker host and the API must exec into `cursor-agent`.

---

## Live now (2026-11-01)

`GET /health` and `POST /system/health` are live. `/v1` is specified below and not built. The `/v1` Caddy restriction (Stella + `194.126.177.181`) and bearer token are **planned, not live**.

### DNS

A record `api.edison` → `178.105.203.54` created via Hetzner Cloud API from Stella (zone `567656`, TTL 300). Token from `/opt/services/caddy/.env` on Stella. Verified with `dig @8.8.8.8`. Do not write the token into docs.

### Layout

```
/opt/services/
├── docker-compose.yml    # caddy + host-api, network edison-net (external)
├── caddy/
│   ├── Dockerfile        # identical to Stella: xcaddy + caddy-dns/hetzner/v2
│   ├── Caddyfile
│   └── .env              # HETZNER_API_TOKEN, copied from Stella via scp -3, chmod 600
└── host-api/
    ├── Dockerfile        # python:3.13-slim + openssh-client + docker CLI (from docker:29-cli)
    ├── requirements.txt  # fastapi 0.120.*, uvicorn[standard] 0.38.*
    ├── allowed_signers   # copied from Stella /opt/services/health-api/allowed_signers
    └── app/
        ├── __init__.py
        ├── auth.py       # ssh-keygen -Y verify, namespace deploy, principal ddashboard, 60s replay window
        ├── health.py     # snapshot checks
        └── main.py       # GET /health, POST /system/health
```

| Container | `container_name` | Ports | Notes |
|-----------|------------------|-------|-------|
| `caddy` | set | publishes **443 only** | Port 80 is not published. `:80 HTTP/2 skipped` log lines are harmless. |
| `host-api` | set | none | Docker socket mounted to inspect containers and `docker exec` into `cursor-agent`. |

Both join `edison-net`.

### Endpoints

| Method | Path | Auth | Purpose |
|---|---|---|---|
| GET | `/health` | none | Liveness `{"status":"ok"}` |
| POST | `/system/health` | signed `system-health`, `allowed_signers` | Host snapshot for Atlas |

Same SSH-signature scheme as Stella health-api and the Atlas host API (`ssh-keygen -Y sign` / `-Y verify`, namespace `deploy`, principal `ddashboard`, 60 s replay window). See [`../stella-server/health-api.md`](../stella-server/health-api.md) and [`../atlas/host-api.md`](../atlas/host-api.md).

### Snapshot fields

| Key | Contents |
|---|---|
| `host` | `edison` |
| `timestamp` | ISO timestamp |
| `docker_daemon` | `{status, version}` |
| `containers` | `{name: {status, started_at, restart_count}}`. Core list: `caddy`, `host-api`, `cursor-agent`. Missing → `"missing"`. |
| `agent` | `{status, version}` via `cursor-agent --version` inside the container |
| `mounts` | `{path: {status}}`. `EDISON_EXPECTED_MOUNTS`, default `/mnt/osgar.datahub.foxcraft.digital`. Checks mountpoint + `ls` with 8 s timeout. |
| `certs` | `{domain: {expires, days_left, status}}` via TLS to `caddy:443` with SNI |
| `disk` | `{total_gb, free_gb, used_pct}` (container root = host disk) |
| `cpu_ram` | `{cpu_threads, load_*, ram_*}` from `/proc` |

### Caddyfile

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
    reverse_proxy host-api:8080
}
```

`@blocked` = not Atlas prod / Stella / MacBook → 403. `@system_blocked` = path `/system/*` and not Atlas prod / Stella → 403. Then `reverse_proxy host-api:8080`.

Planned, not live: further restrict `/v1/*` to Stella (`95.217.144.93`) + `194.126.177.181`, and require `Authorization: Bearer <EDISON_HOST_API_TOKEN>`.

### allowed_signers

Two `ddashboard` lines. Same keys as Stella health-api.

| Comment | Who |
|---------|-----|
| `foxcraft-digital@dedi6724` | atlas-prod |
| (none) | atlas-dev |

### Tests (2026-11-01)

| From | Call | Result |
|------|------|--------|
| MacBook | `GET /health` | `{"status":"ok"}` |
| MacBook | `POST /system/health` | **403** (IP not allowed). Correct. |
| Stella, atlas-dev key (`/opt/apps/laravel/atlas.dev.foxcraft.digital/deploy-key/id_ed25519`) | signed `POST /system/health` | Full snapshot. |

Snapshot values at **2026-11-01 02:56 UTC:** Docker 29.9.0, all 3 containers up, cursor-agent `2026.10.01-e373342`, osgar mount up, cert valid until Jan 30 2027 (89 days), disk 9.9 %, load 0.52, RAM 10.1 %.

**UNVERIFIED:** signed call from Atlas prod (`2.31.42.13`) with the atlas-prod key.

### Gotchas

- After `rm` + `nano` of the Caddyfile use `docker compose up -d --force-recreate caddy` (new inode; `restart` keeps the old file). Same class as Stella Issue 4.
- First build of the Caddy image took several minutes (`go build` prints nothing while compiling on the CX33).

---

## Role

```
Browser  →  Edison web app (Laravel, Stella)
                POST /api/chats/{id}/messages   (SSE to the browser)
                     ↓
                POST {EDISON_HOST_API_URL}/v1/runs
                     ↓
            Edison host API  (this spec)
                     ↓
            cursor-agent-run | claude  in /mnt/<host>
```

The web app is agent-agnostic. Switching Cursor → Claude is `EDISON_AGENT=claude` plus a host implementation. Do not add vendor CLIs to the Laravel repo.

---

## What the web app already calls

| When | Method | Path | Accept |
|---|---|---|---|
| User sends a message (UI) | `POST` | `/v1/runs` | `text/event-stream` |
| JSON fallback / tests | `POST` | `/v1/runs` | `application/json` |

The app **does not** call abort or health yet. Build them anyway so the host is complete.

Connect timeout: 5 s. Read timeout: `EDISON_AGENT_TIMEOUT` (default **180 s**). If `EDISON_HOST_API_TOKEN` is set, the app sends `Authorization: Bearer <token>`.

**Not built.** Next step before implementing `/v1/runs`: verify Cursor `stream-json` field names with a test run.

---

## Required endpoints

`/v1` below is the contract for the Edison app. **Not built.** Live liveness is `GET /health` (above), not `GET /v1/health`.

### `GET /v1/health`

Liveness for Docker / Caddy / uptime. No auth.

**200**

```json
{ "status": "ok", "agents": ["cursor"] }
```

`agents` is the list this process can actually exec. Omit Claude until it is installed.

---

### `POST /v1/runs`

Start or resume a run. **This is the only endpoint the web app needs to go live.**

**Headers**

| Header | Required | Notes |
|---|---|---|
| `Content-Type` | yes | `application/json` |
| `Accept` | yes | `text/event-stream` when `stream` is true; `application/json` otherwise |
| `Authorization` | if the host requires a token | `Bearer <EDISON_HOST_API_TOKEN>` |

**Body**

```json
{
  "prompt": "Fix the ingest timeout",
  "workspace": "/mnt/osgar.datahub.foxcraft.digital",
  "resume": null,
  "stream": true,
  "agent": "cursor"
}
```

| Field | Type | Required | Meaning |
|---|---|---|---|
| `prompt` | string | yes | User text. 1–20000 characters (app already validates). |
| `workspace` | string | yes | Absolute path **on the host**. Must be an allowed sshfs mount. |
| `resume` | string \| null | no | Previous `session_id` for this chat. `null` = new session. |
| `stream` | boolean | no | Default `false`. UI always sends `true`. |
| `agent` | string | yes | `cursor` now. `claude` later. Unknown value → **400**. |

**Workspace allowlist** (must match the app catalog in `config/edison.php`):

| Mount | Repo | Mounted today |
|---|---|---|
| `/mnt/osgar.datahub.foxcraft.digital` | osgar.datahub | yes |
| `/mnt/advoapp.finditoo.foxcraft.digital` | advoapp.finditoo | no |
| `/mnt/dominikliss.foxcraft.digital` | dominikliss | no |
| `/mnt/atlas.dev.foxcraft.digital` | atlas.dev | no |
| `/mnt/hypershape.foxcraft.digital` | hypershape | no |
| `/mnt/edison.dev.foxcraft.digital` | edison.dev | no |

Reject anything else (path traversal, `..`, relative paths, `/`, `$HOME`). **404** if the mount is missing or not mounted. **Do not** create mounts from the API.

---

#### Streaming response (`stream: true`)

**200** + `Content-Type: text/event-stream` + `Cache-Control: no-cache` + `X-Accel-Buffering: no`.

Preferred shape (what the Laravel parser is written for):

```
event: delta
data: {"text":"I'll look through "}

event: delta
data: {"text":"osgar.datahub.\n"}

event: done
data: {"output":"I'll look through osgar.datahub.\n…","session_id":"sess_01H…"}
```

Rules:

- Flush every `delta`. Do not buffer the whole run.
- `text` is an increment, not the full reply so far.
- `done.output` is the full assistant text. The app concatenates deltas; it uses `output` only if it received no deltas.
- `session_id` is required on `done`. The app stores it on the chat and sends it back as `resume`.
- One blank line after each event (`\n\n`).
- If the client drops, stop the CLI (SIGTERM, then SIGKILL).

**Also accepted** (same parser):

NDJSON:

```
{"type":"delta","text":"Hello "}
{"type":"done","output":"Hello world","session_id":"sess_01H…"}
```

Cursor CLI `stream-json` (map it — do not forward raw tool events unless you flatten them to `text`):

```
{"type":"assistant","message":{"content":[{"type":"text","text":"Hello "}]}}
{"type":"result","result":"Hello world","session_id":"sess_01H…"}
```

---

#### JSON response (`stream: false`)

**200** + `Content-Type: application/json`

```json
{
  "output": "I'll look through osgar.datahub.\n…",
  "session_id": "sess_01H…"
}
```

The app treats this as a single chunk. Useful for tests and for a host that cannot stream yet.

---

### `POST /v1/runs/abort`

Stop an in-flight run. The web app does not call this yet; implement it so a Stop button can land without another host change.

**Body**

```json
{ "session_id": "sess_01H…", "agent": "cursor" }
```

**200**

```json
{ "status": "aborted", "session_id": "sess_01H…" }
```

**404** if the session is unknown or already finished. Killing a finished session is not an error if you prefer **200** `{ "status": "idle" }` — pick one and keep it.

---

## Errors

Always JSON `{ "message": "…" }` except on an open SSE stream (then send a last event and close).

| Status | When |
|---|---|
| 400 | Missing/invalid field, unknown `agent`, workspace not in the allowlist |
| 401 | Missing or bad bearer token |
| 403 | Caddy IP allowlist (`@blocked` / `@system_blocked`) |
| 404 | Workspace not mounted, unknown `session_id` on abort |
| 409 | Session already running (same `resume` still in flight) |
| 502 | CLI crashed or produced no usable output |
| 504 | CLI exceeded the host’s own deadline |

On a stream that has already started, emit then close:

```
event: done
data: {"output":"<text so far>","session_id":"sess_01H…"}
```

or, if you prefer an explicit failure:

```
event: error
data: {"message":"cursor-agent exited 1"}
```

The current Laravel parser treats unknown SSE names with a `text` field as deltas. Prefer `done` with partial `output` so the chat still saves.

---

## Auth and network

Live today (`/health`, `/system/*`):

- Bind on `edison-net`. Caddy is the only published ingress (443).
- `/system/*` is SSH-signed (`system-health`) plus Caddy IP allowlist (Atlas prod + Stella).
- Do not expose the API to the public internet.

Planned for `/v1/*` (not live):

- Require a bearer token (`EDISON_HOST_API_TOKEN` on the app, same secret on the host).
- Caddy-restrict to Stella (`95.217.144.93`) + `194.126.177.181`.
- This token is **not** a ddashboard user token. Users never call the host API.

---

## Agent backends

`agent` selects the process. The HTTP contract stays the same.

### `cursor` (now)

Working directory: `workspace`.

```bash
cursor-agent-run -p --trust --force --output-format stream-json --resume "$RESUME" "$PROMPT"
```

Notes from [cursor-agent.md](cursor-agent.md):

- Use **`cursor-agent-run`** (umask `002`), never raw `cursor-agent`, if the run may write files.
- `--trust` is required in a new directory.
- `--force` is required for print mode to write files.
- `--resume <session_id>` continues the chat. Omit on first turn.
- Map `stream-json` assistant text → SSE `delta`. Map the final result / session → `done`.
- Default model is still **TODO**. Set `--model` explicitly once known.
- Unattended prod should use `CURSOR_API_KEY`, not an interactive login.
- Do not run `git checkout`, `switch`, `stash`, `reset`, or `commit` on the live staging tree. The API should pass that as a system/instruction constraint. A separate worktree is still an open TODO.

### `claude` (later)

Same `POST /v1/runs` body. Host execs Claude Code in `workspace` and maps its stream onto the same `delta` / `done` events. Until then, `agent: "claude"` → **400** `{ "message": "Agent claude is not installed." }`.

---

## Sessions

| Concept | Owner | Id |
|---|---|---|
| Chat in the web app | Laravel DB | integer `chats.id` |
| Vendor CLI session | Host / CLI | string `session_id` |

The host must:

1. Create a `session_id` on the first run (`resume` null).
2. Return it on every `done` / JSON body.
3. Resume the same CLI session when `resume` is that id + the same `agent` + same `workspace`.
4. Reject resume across workspaces or agents (**400**).

Prefix ids so they never collide (`cursor_…`, later `claude_…`).

---

## Concurrency

Assume **one run per `session_id`**. A second `POST /v1/runs` with the same `resume` while the first is live → **409**. Different chats (different sessions) may run in parallel, but Cursor headless has been reported at a low process cap (~3). Queue or reject extras with **429**.

---

## Recommended extras (not required to connect the app)

| Method | Path | Why |
|---|---|---|
| `GET` | `/v1/agents` | `{ "current": "cursor", "available": ["cursor"] }` |
| `GET` | `/v1/workspaces` | Mounts the host actually has, for ops — the app already has its own catalog |

Do **not** add chat CRUD, auth, or repo lists to this API. That lives in Laravel.

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
data: {"output":"Ingest starts in `Program.cs`.\n","session_id":"cursor_01JK…"}
```

Follow-up from the same chat sends `"resume": "cursor_01JK…"`.

---

## Implementation sketch

One Docker service (`host-api`) on `edison-net`. Health is live. `/v1` is not built.

When `/v1` lands in the same process:

1. Validate token, body, workspace, agent.
2. If `resume` is set, attach to that process/session; else start one.
3. Exec the CLI in `workspace` with stdout piped (`docker exec` into `cursor-agent`).
4. Translate vendor events → SSE `delta` / `done`.
5. On client disconnect or `POST /v1/runs/abort`, kill the process group.
6. Never persist user prompts beyond what the CLI needs for `--resume`.

Disable proxy buffering (Caddy `flush_interval -1` on the `/v1` path).

---

## Out of scope

- ddashboard login
- Chat history (Laravel `chats` / `chat_messages`)
- Repo catalog UI
- Running builds/tests inside the Edison container (no SDK there — SSH to Stella if needed)
- Writing Atlas or Stella features into this API
