# .NET App Deployment Pattern

Established 2026-08-06 with `finditoo-advoapp` (`advoapp`) as the first implementation. This is the template for any future .NET app on Stella.

## Three environments, three different mechanisms

| Environment | Where | How | URL |
|---|---|---|---|
| **Dev** | Stella, Docker | `dotnet watch` hot-reload, bind-mounted source | `advoapp.finditoo.foxcraft.digital` (currently — see note below) |
| **Staging** | — | Removed 2026-08-10 (was built but never Caddy-routed) | — |
| **Production** | Azure App Service | Clean checkout → `dotnet publish` → zip → `az webapp deploy` | Azure-assigned `*.azurewebsites.net` (custom domain not yet configured) |

**⚠️ Current state note:** The original plan was dev + staging as two separate Docker containers, with staging (the built, non-watching container) exposed at `advoapp.finditoo.foxcraft.digital`. In practice, Dominik asked for that public subdomain to show the **live dev container** instead, since "the production URL is completely different" (i.e., Azure is production, so the Stella subdomain only ever needs to reflect current dev work). As of now, the Caddyfile routes `advoapp.finditoo.foxcraft.digital` → `advoapp-dev:8080`. The unused staging container/files were removed 2026-08-10 (see below).

---

## Folder layout

```
/opt/apps/dotnet/
  ├── advoapp.finditoo.foxcraft.digital/            ← dev container definition
  │     ├── src/                                     ← git clone; live-edited via the `-ssh` container (see [`dev-ssh-access.md`](dev-ssh-access.md))
  │     ├── Dockerfile.dev                             ← dev build (SDK image + supervisor)
  │     ├── Dockerfile.ssh                             ← sshd for Cursor Remote-SSH
  │     ├── supervisord.conf                           ← bind-mounted into `-dev` (not baked into the image)
  │     ├── authorized_keys                            ← both devs' public keys
  │     └── docker-compose.dev.yml                     ← `-dev` + `-ssh` services (bind-mounts src/)
  │
  └── advoapp-production.finditoo.foxcraft.digital/  ← clean checkout for Azure deploys only
        └── src/                                       ← git clone, branch `master`, reset --hard on every deploy
```

Note: despite the `.finditoo.foxcraft.digital`-style naming, the production folder does **not** correspond to a real public subdomain — it's just naming convention carried over for consistency/findability on disk. If a real `advoapp-production.finditoo.foxcraft.digital` custom domain is ever wanted, that would need to be configured as an Azure custom domain (TXT + CNAME records, requires Basic+ App Service plan — Free tier doesn't support custom domains) rather than a Caddy site block, since production traffic terminates at Azure, not Stella.

---

## Dev container — hot reload

`Dockerfile.dev`:
```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:10.0
RUN apt-get update && apt-get install -y --no-install-recommends supervisor \
    && rm -rf /var/lib/apt/lists/*
WORKDIR /src
EXPOSE 8080
```

`docker-compose.dev.yml`:
```yaml
services:
  advoapp-dev:
    build:
      context: .
      dockerfile: Dockerfile.dev
    container_name: advoapp-dev
    working_dir: /src
    volumes:
      - ./src:/src
      - ./supervisord.conf:/etc/supervisor/conf.d/app.conf
    ports:
      - "127.0.0.1:5080:8080"
    environment:
      - ASPNETCORE_URLS=http://+:8080
      - ASPNETCORE_ENVIRONMENT=Development
      - DOTNET_USE_POLLING_FILE_WATCHER=1
    command: ["supervisord", "-c", "/etc/supervisor/conf.d/app.conf"]
    restart: unless-stopped
    networks:
      - edge
      - default

  advoapp-ssh:
    build:
      context: .
      dockerfile: Dockerfile.ssh
    container_name: advoapp-ssh
    volumes:
      - ./src:/home/dominik/app
      - ./src:/home/pawel/app
    ports:
      - "2202:22"
    restart: unless-stopped
    # No networks: key — must stay off `edge`. See dev-ssh-access.md.

networks:
  edge:
    external: true
```

Key details:
- `DOTNET_USE_POLLING_FILE_WATCHER=1` — required. Native filesystem events (inotify) don't reliably cross into containers via bind mounts on this setup; without polling, `dotnet watch` silently never notices saved changes.
- `127.0.0.1:5080` binding kept for direct Cursor port-forwarding as a fallback, in addition to the Caddy route.
- To view live changes: edit in Cursor (connected via Remote-SSH to the **app's `-ssh` container**, not the Stella host — see [`dev-ssh-access.md`](dev-ssh-access.md)) → save → `dotnet watch` in the `-dev` container recompiles/hot-patches automatically within a couple seconds → refresh browser at `https://advoapp.finditoo.foxcraft.digital`.
- Harmless log line to ignore: `dotnet watch ❌ Failed to launch ... Unable to launch the browser` — `dotnet watch` tries to auto-open a browser on the host, which doesn't exist inside the container. Not an error affecting the app itself.
- Start it: `docker compose -f docker-compose.dev.yml up -d`
- **`restart: unless-stopped` added 2026-08-10.** Originally missing — `advoapp-dev` was killed by a Docker daemon restart on 2026-08-07 (unrelated troubleshooting elsewhere on the server) and, with no restart policy, silently stayed down for **3 days** before being noticed. Nothing was monitoring it at the time. Root cause confirmed via `docker inspect --format '{{.State.FinishedAt}}'` cross-referenced against `journalctl -u docker` daemon-restart timestamps — not an OOM kill (`OOMKilled: false`, no kernel log entry), just a daemon bounce with no policy to bring the container back. This exact scenario is now also caught automatically by [`health-api`](health-api.md), Atlas polling permitting.

`supervisord.conf` (bind-mounted, not baked into the image — same structure as [`apps/osgar-datahub/setup.md`](apps/osgar-datahub/setup.md), without a `[program:scss]` block because advoapp has no SCSS build step):
```ini
[supervisord]
nodaemon=true
logfile=/dev/stdout
logfile_maxbytes=0

[inet_http_server]
port=*:9001
username=agent
password=<see .env or ask Dominik>

[rpcinterface:supervisor]
supervisor.rpcinterface_factory = supervisor.rpcinterface:make_main_rpcinterface

[program:app]
command=/bin/bash -c "(pkill -f [f]inditoo.advoapp || true) && sleep 1 && exec dotnet watch run --urls http://+:8080 --non-interactive"
directory=/src
autostart=true
autorestart=true
startretries=3
stopasgroup=true
killasgroup=true
stdout_logfile=/dev/stdout
stdout_logfile_maxbytes=0
redirect_stderr=true

[supervisorctl]
serverurl=http://127.0.0.1:9001
username=agent
password=<same as above>
```

> **Migrated to supervisord 2026-09-07** — `advoapp-dev` now matches `osgar-datahub-dev`'s pattern exactly (minus the SCSS watcher). Reason: close the gap tracked in [`apps/osgar-datahub/setup.md`](apps/osgar-datahub/setup.md)'s "Open follow-ups" — orphaned `dotnet watch` process chains without `stopasgroup`/`killasgroup`, and no restart trigger from the `-ssh` container without Docker socket access. Verified 2026-09-07 via `docker exec advoapp-dev ps aux` showing exactly one `supervisord` → `dotnet watch` → `dotnet-watch.dll` → `dotnet run` → `finditoo.advoapp` chain. **`stopasgroup`/`killasgroup` did not fully prevent orphans** — leftover `dotnet run` → `finditoo.advoapp` children reparented to PID 1 recurred **2026-10-09** and **2026-10-12**. On 2026-10-12 that orphan was a **side effect** of a crash after a `-ssh` container build baked the wrong paths into `bin/`/`obj/` (see Troubleshooting below), not a supervisor-group-kill failure on its own.

**`[program:app]` command wrapper added 2026-10-12** — leftover app processes are killed on every (re)start of `app`. Gotchas (all hit while writing the line):
- supervisord treats ` ;` as the start of an inline comment. Use `&&` / `||`, never `;`.
- Nested single quotes inside the double-quoted `bash -c` string break parsing. The `[f]inditoo` bracket pattern avoids quoting `finditoo.advoapp` and also avoids `pkill` matching its own command line.
- `|| true` is required — `pkill` exits 1 when nothing matches, and without it the `&&` chain never reaches `dotnet watch`.
- This only helps when `dotnet watch` **exits** (supervisord then restarts `app`, which runs `pkill` first). It does **not** clean up while `dotnet watch` sits at `Waiting for a file to change before restarting ...`.

Run manually (start it, view logs): `docker compose -f docker-compose.dev.yml up`

### Troubleshooting: app crashes at startup after a build from the -ssh container

Confirmed **2026-10-12** (same day as the orphan loop below — this is the root cause; the port bind failure is the leftover).

**Symptom:** the app is stale or returns `502`. `supervisorctl status` shows `app RUNNING`. `docker logs advoapp-dev` ends with `Unhandled exception. System.IO.DirectoryNotFoundException: /home/pawel/app/wwwroot/` (user can also be `dominik`), then `dotnet watch ❌ Exited with error code 134` and `Waiting for a file to change before restarting ...`.

**Secondary symptom:** `address already in use` on 8080 loops in the log because an orphaned `dotnet run` → `finditoo.advoapp` chain (PPID 1) still holds the port. This is a **side effect**, not the root cause.

**Root cause:** `src/` is bind-mounted into both containers at different paths (`/src` in `advoapp-dev`, `/home/<user>/app` in `advoapp-ssh`). A build run from the `-ssh` container (Cursor C# extension, build-on-open, manual `dotnet build` / `dotnet run`) bakes the `-ssh` paths into `bin/` and `obj/` (static web assets manifests, `FileListAbsolute.txt`). `advoapp-dev` then crashes at startup because it can't find that path.

**Diagnosis** — run on the Stella host:
```bash
docker logs --tail 40 advoapp-dev
docker exec advoapp-dev ps -eo pid,ppid,etime,args --forest
cd /opt/apps/dotnet/advoapp.finditoo.foxcraft.digital/src && grep -rl "/home/" bin obj 2>/dev/null | head
```

Files under `bin/`/`obj/` containing a `/home/<user>/app` path confirm this cause.

**Fix:**
```bash
cd /opt/apps/dotnet/advoapp.finditoo.foxcraft.digital/src
rm -rf bin obj
docker restart advoapp-dev
```

Wait ~75s. Expect one chain `supervisord` → `dotnet watch` → `dotnet-watch.dll` → `dotnet run` → `finditoo.advoapp` and HTTP `302` on `https://advoapp.finditoo.foxcraft.digital/`.

**Prevention:** never build or run from the `-ssh` container — only `advoapp-dev` builds. Check the C# extension settings in Cursor Remote-SSH sessions. Add to `.cursor/rules/dev-environment.mdc` if not already there.

### Troubleshooting: stuck dev container (orphaned dotnet chain)

Same incident twice: **2026-10-09** and **2026-10-12**. On 2026-10-12 the orphan was the leftover after the `-ssh` path crash above.

**Symptom:** the app serves a stale build or doesn't pick up changes. Supervisor shows `app RUNNING`, and `curl` on `https://advoapp.finditoo.foxcraft.digital/` can still return `302` (served by the stale orphan, so this is **not** a health signal).

**Cause:** a previous `dotnet watch` died and left its `dotnet run` → `finditoo.advoapp` children behind, reparented to PID 1. They keep holding port 8080. The current `dotnet watch` then fails with `System.IO.IOException: Failed to bind to address http://[::]:8080: address already in use` (exit code 134) and sits at `Waiting for a file to change before restarting ...`.

**⚠️ On 2026-10-12 the original `dotnet watch` death was the `DirectoryNotFoundException` from a `-ssh` build (see above).** The 2026-10-09 first death is still unexplained. The 2026-10-12 `pkill` wrapper only runs when `app` (re)starts — it does not help while `dotnet watch` is sitting in the wait loop. `stopasgroup`/`killasgroup` (added 2026-09-07) did not prevent the leftover children.

**Diagnosis** — run on the Stella host, not in `advoapp-ssh`:
```bash
docker exec advoapp-dev supervisorctl -c /etc/supervisor/conf.d/app.conf status
docker logs --tail 80 advoapp-dev
docker exec advoapp-dev ps -eo pid,ppid,etime,args --forest
docker logs --since 6h advoapp-dev | grep -iE "exited|error code|FATAL|Failed to bind"
```

- `supervisorctl` is not installed on the host, so it has to go through `docker exec`.
- advoapp logs go to stdout (no `app-logs/` mount like osgar-datahub), so use `docker logs`, not the `-ssh` container.
- **Healthy:** exactly one chain `supervisord` → `dotnet watch` → `dotnet-watch.dll` → `dotnet run` → `finditoo.advoapp`.
- **Unhealthy:** a second `dotnet run` / `finditoo.advoapp` chain with PPID 1 and a much older `etime`.

**Fix:**
```bash
docker restart advoapp-dev
```

Wait ~60s, re-run the `ps` command, and confirm a single fresh chain plus normal request logs.

Lighter alternative: `docker exec advoapp-dev kill <orphan dotnet run PID> <orphan finditoo.advoapp PID>`, then touch a source file. A full restart is cleaner.

---

## Staging container — removed 2026-08-10

Previously existed as a second Dockerfile/compose pair in the same folder as the dev container (`Dockerfile` + `docker-compose.yml`, vs. dev's `Dockerfile.dev` + `docker-compose.dev.yml`). Built during initial setup per the original three-environment plan, but never actually routed to by Caddy after the decision to point the public subdomain at the live dev container instead (see the "Current state note" above). Confirmed unused — not in `docker ps -a`, no Caddy block referenced it, and the production deploy pipeline (`deploy-advoapp.sh`) builds independently from a separate clean checkout, never touching this container at all.

Removed entirely: container, built image, and both defining files (`Dockerfile`, `docker-compose.yml`). The `advoapp.finditoo.foxcraft.digital` folder now contains `Dockerfile.dev`, `Dockerfile.ssh`, `supervisord.conf`, `authorized_keys`, `docker-compose.dev.yml`, and `src/` — matching what's actually deployed.

If a genuine pre-deploy build-sanity-check is ever wanted, it should get its own subdomain and Caddy route rather than being silently built-but-unrouted — otherwise it becomes indistinguishable from dead weight, as it did here.

---

## Production — Azure App Service

### Service principal (Azure AD)

- App registration: `stella-deploy-advoapp` (single-tenant)
- Client secret generated (has an **expiry** — check Certificates & secrets in the app registration for the date; rotate before it lapses or production deploys will fail with `invalid_client`/`Invalid client secret provided`)
- Role assignment: **Website Contributor**, scoped to just the one App Service (not subscription-wide) via the App Service's own **Access control (IAM)** blade — least-privilege, so a leaked credential can only touch this one app

### Credentials storage

`/opt/services/azure-deploy/.env` (chmod 600, not committed):
```
AZURE_TENANT_ID=...
AZURE_CLIENT_ID=...
AZURE_CLIENT_SECRET=...
AZURE_SUBSCRIPTION_ID=...
AZURE_RESOURCE_GROUP=finditoo-advo-app
AZURE_WEBAPP_NAME=finditoo-advo-app
```

**Common setup mistakes (both hit during initial setup, worth knowing):**
- **`invalid_tenant`** — usually means Client ID and Tenant ID got swapped (both are GUIDs, sit next to each other on the app registration Overview page).
- **`Invalid client secret provided... not the client secret ID`** — Azure shows a "Secret ID" column that looks like a credential but isn't; only the "Value" column (visible once, at creation time only) is the actual secret.

### The deploy script

`/opt/services/azure-deploy/deploy-advoapp.sh`:
```bash
#!/bin/bash
set -euo pipefail

source /opt/services/azure-deploy/.env

SRC_DIR="/opt/apps/dotnet/advoapp-production.finditoo.foxcraft.digital/src"
PUBLISH_DIR="/opt/apps/dotnet/advoapp-production.finditoo.foxcraft.digital/publish"
ZIP_PATH="/opt/apps/dotnet/advoapp-production.finditoo.foxcraft.digital/publish.zip"
LOG_PATH="/opt/services/azure-deploy/logs/deploy.log"

mkdir -p "$(dirname "$LOG_PATH")"
log() { echo "$(date -u +%Y-%m-%dT%H:%M:%SZ) $1" >> "$LOG_PATH"; }

echo "1/5 — Pulling latest master..."
cd "$SRC_DIR"
git fetch origin master
git reset --hard origin/master
log "pulled $(git rev-parse HEAD)"

echo "2/5 — Publishing (dotnet publish)..."
rm -rf "$PUBLISH_DIR"
docker run --rm \
  -v "$SRC_DIR:/src" \
  -v "$PUBLISH_DIR:/publish" \
  -w /src \
  mcr.microsoft.com/dotnet/sdk:10.0 \
  dotnet publish -c Release -o /publish

echo "3/5 — Zipping..."
rm -f "$ZIP_PATH"
cd "$PUBLISH_DIR"
zip -r "$ZIP_PATH" . > /dev/null

echo "4/5 — Authenticating to Azure..."
az login --service-principal \
  -u "$AZURE_CLIENT_ID" \
  -p "$AZURE_CLIENT_SECRET" \
  --tenant "$AZURE_TENANT_ID" \
  --output none
az account set --subscription "$AZURE_SUBSCRIPTION_ID"

echo "5/5 — Deploying to Azure App Service..."
az webapp deploy \
  --resource-group "$AZURE_RESOURCE_GROUP" \
  --name "$AZURE_WEBAPP_NAME" \
  --src-path "$ZIP_PATH" \
  --type zip

az logout

log "deploy succeeded"
echo "Done — deployed to $AZURE_WEBAPP_NAME"
```

**⚠️ Branch name: `master`, not `main`.** The repo's default branch is `master`. This tripped up the first setup attempt (`fatal: couldn't find remote ref main`) — check `git ls-remote --heads <repo>` before assuming `main` for any future repo.

**Why `dotnet publish` runs via `docker run` rather than a native SDK install:** keeps Stella's host free of language toolchains — same principle as everything else on this server. The container is ephemeral (`--rm`), just borrows the official SDK image for the duration of the build.

**Host dependencies this script needs directly on Stella (not containerized):** `zip` (`apt install zip`), `az` CLI (`curl -sL https://aka.ms/InstallAzureCLIDeb | bash`), Docker itself.

### Manual run
```bash
/opt/services/azure-deploy/deploy-advoapp.sh
```
Takes several minutes end-to-end — `dotnet publish` inside Docker, then Azure's own "warming up Kudu" + site restart cycle, which alone commonly takes 1–3 minutes on top of the build.

### Remote trigger

See [`deploy-api.md`](deploy-api.md) — **Atlas** triggers this script via the signed deploy-api (`stella_deploy_api` config). Do not SSH in manually except as an emergency fallback. Atlas-side: [`../atlas/dotnet-azure.md`](../atlas/dotnet-azure.md).

---

## New .NET-dev domain — full checklist

Reference run: `osgar.datahub.foxcraft.digital` (Blazor Server, .NET 10). Complements the patterns above and the Caddy notes in [`infrastructure.md`](infrastructure.md).

1. **App folder + source** under `/opt/apps/dotnet/<subdomain>/src` — clone the repo, or for an empty repo: `dotnet new blazor -n <Name> --interactivity Server`
2. **`Dockerfile.dev` + `docker-compose.dev.yml`** after the `advoapp-dev` pattern above. Adjust the host port if `5080` is already taken (e.g. `127.0.0.1:5081:8080`)
3. **Per-app `-ssh` container** — `Dockerfile.ssh`, `authorized_keys`, compose service with **no `networks:` key**, next free `22XX` port, `DOCKER-USER` rules, host `src/` group perms (**both** the one-time `chgrp`/`chmod`/SetGID fix **and** the `setfacl` default-ACL fix). Full checklist: [`dev-ssh-access.md`](dev-ssh-access.md)
4. **DNS A-Record** via Hetzner Cloud API (see [`infrastructure.md`](infrastructure.md) — zone `567656`)
5. **Caddy block** with explicit Let's Encrypt production `dir` pin and `@blocked` deny-list for any currently-active client IPs (see [`infrastructure.md`](infrastructure.md) and [`client-ip-access.md`](client-ip-access.md)):
   ```caddyfile
   <subdomain>.foxcraft.digital {
       tls {
           issuer acme {
               email projects@foxcraft.digital
               dir https://acme-v02.api.letsencrypt.org/directory
               dns hetzner {env.HETZNER_API_TOKEN}
           }
       }
       @blocked remote_ip <client-ip-1> <client-ip-2>   # omit if no client IPs are active
       respond @blocked 403
       reverse_proxy <container-name>:8080
   }
   ```
6. **`docker compose restart caddy`** from `/opt/services` (not `up -d` alone). Watch issuance logs; if stuck, diagnose/delete stale `_acme-challenge` TXT per infrastructure Issue 2, then restart again
7. **`docker compose -f docker-compose.dev.yml up -d`** in the app folder
8. **`curl -sI https://<subdomain>`** — `502` for the first seconds/minutes after start is normal while `dotnet watch` restores/builds
9. **Health + backup** — new containers are discovered automatically. Opt out with `health.enable=false` / `backup.exclude=true` if needed; optional `health.http=<url>` (edge only). See [`health-api.md`](health-api.md) and [`backup.md`](backup.md).

Do **not** omit `dir` — without it Caddy can fall back to ZeroSSL or Let's Encrypt staging; staging certs are untrusted and leave the domain effectively broken (details in infrastructure Issue 3).

### Checklist addition — restart policy is not optional

Every service in every `docker-compose*.yml` on this server should have `restart: unless-stopped` unless there's a specific reason not to (e.g. a genuinely one-shot job). This was missed on both `advoapp-dev` and `osgar-datahub-dev` when they were first created — both used the same template, both had the same gap, and the first one went unnoticed for 3 days. Check for this explicitly in step-by-step reviews of any new `.dev.yml`.

---

## Open follow-ups

- [ ] **2026-10-09 `dotnet watch` death still unexplained.** 2026-10-12 was a `-ssh` path bake (`DirectoryNotFoundException`); the leftover orphan is a side effect. The `pkill` wrapper only runs when `app` (re)starts, not while `dotnet watch` sits in the wait loop. Consider a watchdog or health check for duplicate `dotnet` chains — a container-level check cannot see this state (see [`health-api.md`](health-api.md)).
