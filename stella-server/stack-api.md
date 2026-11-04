# stack-api (container management for Atlas)

**Status:** live since 2026-11-04. Docker access via the Engine API (httpx over the Unix socket — no compose CLI in the image). Signed `list` / `get` / `restart` plus an append-only SQLite audit log. No `apply` yet; until then stack-api can only list and restart.

**URL:** `https://stella-stack-api.foxcraft.digital`

**Purpose:** the single entry point through which Atlas creates, changes, restarts and deletes app stacks (WordPress, Laravel, .NET) on Stella. Atlas holds the spec; stack-api renders compose files from its own blueprints and applies them. Staging from an Atlas backup becomes a `create` + `restore` job here; [`wp-staging-api.md`](wp-staging-api.md) is superseded and will not be deployed as a separate service.

Infra: [`infrastructure.md`](infrastructure.md). System map: [`../integration/system-overview.md`](../integration/system-overview.md).

**Rule (decided 2026-11-04):** stacks under `/opt/apps/` are not edited by hand. Changes go through Atlas → stack-api. Blueprint changes go through the templates in the repo.

---

## Files on Stella

```
/opt/services/stack-api/
  ├── Dockerfile          ← python:3.13-slim + openssh-client
  ├── requirements.txt    ← fastapi, uvicorn, httpx
  ├── .dockerignore       ← .env, allowed_signers, data, images
  ├── .env                ← chmod 600; STACK_API_TOKEN, STACK_API_ALLOWED_IPS, STACK_API_DB
  ├── allowed_signers     ← mounted read-only, not baked into the image
  ├── data/               ← host dir; SQLite at /data/stack-api.sqlite in the container (STACK_API_DB)
  ├── app/main.py
  └── images/ssh-php/     ← shared SSH image (not in the API container)
        ├── Dockerfile
        ├── atlas-sync-users
        └── entrypoint.sh
```

Never write the token value into docs.

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
    read_only: true
    tmpfs:
      - /tmp
    networks:
      - stack
    restart: unless-stopped
```

Top-level network `stack` (Docker name `services_stack`). **Only `stack-api` and `services-caddy-1` are on it.** stack-api is deliberately **not** on `edge`: any container on `edge` could otherwise reach it directly and fake `X-Forwarded-For`. `caddy` has `default`, `edge`, and `stack`.

No published ports.

With the socket mounted, stack-api is **root-equivalent** on Stella. Until `apply` exists it can only list and restart.

---

## Three protection layers

| Layer | Where | Detail |
|---|---|---|
| IP allowlist | Caddy **and** app | Caddy: `@blocked not remote_ip 2.31.42.13 95.217.144.93` → 403. App: rightmost `X-Forwarded-For` must be in `STACK_API_ALLOWED_IPS`; missing header → 403. Applies to every path, including `/health` |
| Secret token | app | Header `X-Stack-Token`, constant-time compare against `STACK_API_TOKEN` (64 hex chars, `openssl rand -hex 32`, min. 32 chars enforced at startup) |
| SSH signature | app | See below |

Every failed check answers a generic `401 {"detail":"unauthorized"}` or `403 {"detail":"forbidden"}`. The reason is only logged (`denied reason=…` / `denied ip=…`).

`2.31.42.13` = Atlas prod. `95.217.144.93` = Stella itself (for Atlas dev). Verified 2026-11-04: a request from a container on Stella to the public URL arrives at Caddy as `95.217.144.93`, not as the Docker gateway.

### Signed requests

```
payload   = "<action>:<unix_ts>:<sha256(request)>"
POST body = {"payload": …, "signature": …, "request": "<JSON string>"}
```

- `ssh-keygen -Y find-principals` (signer → principal), then `ssh-keygen -Y verify -n stack -I <principal>`
- **Namespace `stack`** — not `deploy`. Signatures for deploy-api / health-api are never valid here.
- 60 s window, replay cache on the signature hash (in memory, entries kept 120 s)
- The body hash binds the request JSON to the signature
- `request` must be a JSON object

### Key

| Item | Value |
|---|---|
| Private key | host `atlas`, `/home/atlas/.ssh/atlas-stack-api` (owner `atlas`, no passphrase) |
| Comment | `atlas-stack-api` |
| Principal | `atlas-prod` |
| `allowed_signers` line | `atlas-prod namespaces="stack" ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIBJ7MdsnXmRKl6krZaJYofDeu2GYuap5D0A1b+sCUnej atlas-stack-api` |

Separate from the shared Atlas deploy key (`/home/atlas/.ssh/id_ed25519`, principal `ddashboard`), which is **not** trusted by stack-api.

**Atlas dev:** passes the IP layer, but has **no key** in `allowed_signers` yet. Planned: principal `atlas-dev` with its own key, restricted to stacks marked `sandbox: true`. Do not add the key before that restriction exists in code.

### Atlas `.env`

```
STELLA_STACK_API_URL=https://stella-stack-api.foxcraft.digital
STELLA_STACK_API_TOKEN=<value>
STELLA_STACK_API_ENABLED=false
STELLA_STACK_API_KEY_PATH=
```

Atlas client (2026-11-04): `app/Domain/Stacks/StackApiClient.php`. Flag default **false** — `/api/stacks` is 404 and the nav item is hidden. Setting `stack_api_ssh_key_path` (Settings → GitHub, next to the deploy key path) overrides `STELLA_STACK_API_KEY_PATH`.

| Instance | Enable | Key path |
|---|---|---|
| atlas.dev | `STELLA_STACK_API_ENABLED=true` first | `/var/www/.ssh/id_ed25519` (same file as the atlas-dev deploy key) |
| Atlas prod | keep false until atlas.dev is proven | `/home/atlas/.ssh/atlas-stack-api` |

Admin role only. Atlas records restarts in `stack_actions` (`user_id`, stack, service, result, detail, created_at`). stack-api keeps its own audit log.

### Atlas client

`StellaSshSigner::sign($payload, 'stack', $keyPath)` — namespace **`stack`**, not `deploy`. Payload `<action>:<unix_ts>:<sha256(request_json)>`. POST `{payload, signature, request}` where `request` is the exact JSON string that was hashed. Header `X-Stack-Token`. Restart timeout 65 s.

| Atlas | stack-api |
|---|---|
| `GET /api/stacks` | `POST /stacks/list` action `stack-list` request `{}` |
| `GET /api/stacks/{name}` | `POST /stacks/get` action `stack-get` request `{"name"}` |
| `POST /api/stacks/{name}/restart` | `POST /stacks/restart` action `stack-restart` request `{"name"}` or `{"name","service"}` |

Name `^[a-z0-9][a-z0-9.-]{0,252}$`, service `^[a-z0-9][a-z0-9_-]{0,62}$` — validated in Atlas before the call. Mapped errors: 401/403 → `stack-api rejected the request`; 404 detail passed through; 400 → `invalid input`; 502 → `docker error` (results passed through); connection → `stack-api unreachable`. Never log the token, signature, or key contents.

---

## Stack discovery

A container belongs to a stack if its label `com.docker.compose.project.working_dir` is exactly `/opt/apps/<type>/<name>` with `type` in `{wordpress, laravel, dotnet}` and `name` matching `^[a-z0-9][a-z0-9.-]{0,252}$`. Stack name = folder name, type = parent folder. Everything in `/opt/services` (Caddy, deploy-api, health-api, …) is invisible and cannot be restarted through stack-api.

## Endpoints (current)

Signed endpoints: action = payload prefix. Auth is IP + token + signature unless noted.

| Path | Action | Request | Response |
|---|---|---|---|
| `GET /health` | — | — (IP only) | `{"status":"ok"}` |
| `POST /ping` | `stack-ping` | JSON object | `{"ok":true,"principal":…,"request":{…}}` |
| `POST /stacks/list` | `stack-list` | `{}` | `{"stacks":[{name, type, path, containers:[{id (12 chars), name, service, image, state, status, ports:[{ip, public, private, type}]}]}]}` |
| `POST /stacks/get` | `stack-get` | `{"name"}` | one stack; containers also have `started_at`, `restart_count`, `health`. `400` `invalid name`, `404` `stack not found` |
| `POST /stacks/restart` | `stack-restart` | `{"name", "service"?}` | restarts one service or the whole stack (`db` first); Docker restart `t=10`. `{"name","results":[{service, name, ok}]}`. `404` `service not found`; `502` with `results` if any restart failed |

Docker errors → `502` `"docker error"`.

`ports` includes the bind IP (`ip`, `public`, `private`, `type`) — e.g. `0.0.0.0` and `::` for 2206.

FastAPI docs (`/docs`, `/redoc`, `/openapi.json`) are disabled.

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
    @blocked not remote_ip 2.31.42.13 95.217.144.93
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
| `POST /stacks/list` | Atlas prod | all 6 stacks (`advoapp.finditoo`, `atlas.dev`, `dominikliss`, `edison.dev`, `hypershape`, `osgar.datahub`); no `/opt/services` containers |
| `POST /stacks/get` `hypershape` | Atlas prod | `wp` / `db` / `ssh` with `started_at` / `restart_count` |
| `POST /stacks/restart` hypershape `ssh` | Atlas prod | `ok` |
| name `../services` | Atlas prod | `400` `invalid name`; audit `error` row with stack `../services` |
| name `services` | Atlas prod | `404` `stack not found`; audit `error` row |
| restart service `nope` | Atlas prod | `404`; audit `error` row |
| wrong token on `/stacks/list` | Atlas prod | `401`; audit row (`-`, `denied`, `reason=bad token path=/stacks/list ip=2.31.42.13`) |
| `ports` on list/get | Atlas prod | bind IP present (`0.0.0.0` and `::` for 2206) |
| audit after the first list/get/restart | Stella | 3 `ok` rows; `DELETE FROM audit` failed with `audit is append-only` |

Test helper on host `atlas` (token from Atlas `.env`, sign with `/home/atlas/.ssh/atlas-stack-api`, namespace `stack`):

```bash
stack_call() {
  local ACTION="$1" URLPATH="$2" REQ="$3"
  local STACK_TOKEN TS HASH PAYLOAD TMP BODY
  STACK_TOKEN=$(grep '^STELLA_STACK_API_TOKEN=' /var/www/atlas/.env | cut -d= -f2- | tr -d '[:space:]')
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
REQ='{"hello":"stella"}'
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

## Planned (not built)

Security controls agreed for the Docker-socket phase:

- Strict schema per field (names `^[a-z0-9.-]+$`, PHP versions from a list, whitelisted ini keys, no newlines); commands only as argument lists; every path must resolve under `/opt/apps/<type>/<name>/`
- **Policy check on every rendered compose file before `up`:** refuse `privileged`, `cap_add`, `network_mode: host`, `pid: host`, bind mounts outside the app folder, `docker.sock`, `edge` on `ssh`/`db`/`adminer`, images outside the allowlist
- Mounts limited to `docker.sock`, `/opt/apps`, and the Caddy `sites/` folder; Caddy reload via the admin API, not a container restart
- No Hetzner DNS token in stack-api (wildcard `*.foxcraft.digital` → Stella instead; explicit records keep precedence)
- Soft delete to `/opt/apps/.trash/<name>-<ts>`, volumes kept 7 days
- Jobs in SQLite (not in memory), one lock per stack plus a limit for heavy jobs — job table deferred until long-running actions (`restore`). Audit log is already live.

Build order:

1. ~~Skeleton: auth~~ (done 2026-11-04) → ~~SQLite + `list` / `get` / `restart`~~ (done 2026-11-04; audit log only, no job table yet; jobs come with long-running actions like `restore`)
2. ~~Generic SSH image~~ (done 2026-11-04; `hypershape` adopted by hand) → WP blueprint in stack-api, then move `dominikliss`
3. `apply` with SSH users and PHP settings (`PHP_INI_SCAN_DIR` + per-site `zz-atlas.ini`)
4. `restore` (staging from backup)
5. `php-version`, `delete`, Adminer on demand, then Laravel and .NET blueprints
