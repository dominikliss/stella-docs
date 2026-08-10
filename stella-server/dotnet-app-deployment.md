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
  │     ├── src/                                     ← git clone, live-edited via Cursor/SSH
  │     ├── Dockerfile.dev                             ← dev build (SDK image only, no publish)
  │     └── docker-compose.dev.yml                     ← dev container def (bind-mounts src/, dotnet watch)
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
    ports:
      - "127.0.0.1:5080:8080"
    environment:
      - ASPNETCORE_URLS=http://+:8080
      - ASPNETCORE_ENVIRONMENT=Development
      - DOTNET_USE_POLLING_FILE_WATCHER=1
    command: ["dotnet", "watch", "run", "--urls", "http://+:8080"]
    restart: unless-stopped
    networks:
      - edge
      - default

networks:
  edge:
    external: true
```

Key details:
- `DOTNET_USE_POLLING_FILE_WATCHER=1` — required. Native filesystem events (inotify) don't reliably cross into containers via bind mounts on this setup; without polling, `dotnet watch` silently never notices saved changes.
- `127.0.0.1:5080` binding kept for direct Cursor port-forwarding as a fallback, in addition to the Caddy route.
- To view live changes: edit in Cursor (connected via Remote-SSH to Stella) → save → `dotnet watch` recompiles/hot-patches automatically within a couple seconds → refresh browser at `https://advoapp.finditoo.foxcraft.digital`.
- Harmless log line to ignore: `dotnet watch ❌ Failed to launch ... Unable to launch the browser` — `dotnet watch` tries to auto-open a browser on the host, which doesn't exist inside the container. Not an error affecting the app itself.
- Start it: `docker compose -f docker-compose.dev.yml up -d`
- **`restart: unless-stopped` added 2026-08-10.** Originally missing — `advoapp-dev` was killed by a Docker daemon restart on 2026-08-07 (unrelated troubleshooting elsewhere on the server) and, with no restart policy, silently stayed down for **3 days** before being noticed. Nothing was monitoring it at the time. Root cause confirmed via `docker inspect --format '{{.State.FinishedAt}}'` cross-referenced against `journalctl -u docker` daemon-restart timestamps — not an OOM kill (`OOMKilled: false`, no kernel log entry), just a daemon bounce with no policy to bring the container back. This exact scenario is now also caught automatically by [`health-api`](health-api.md), Atlas polling permitting.

Run manually (start it, view logs): `docker compose -f docker-compose.dev.yml up`

---

## Staging container — removed 2026-08-10

Previously existed as a second Dockerfile/compose pair in the same folder as the dev container (`Dockerfile` + `docker-compose.yml`, vs. dev's `Dockerfile.dev` + `docker-compose.dev.yml`). Built during initial setup per the original three-environment plan, but never actually routed to by Caddy after the decision to point the public subdomain at the live dev container instead (see the "Current state note" above). Confirmed unused — not in `docker ps -a`, no Caddy block referenced it, and the production deploy pipeline (`deploy-advoapp.sh`) builds independently from a separate clean checkout, never touching this container at all.

Removed entirely: container, built image, and both defining files (`Dockerfile`, `docker-compose.yml`). The `advoapp.finditoo.foxcraft.digital` folder now contains only `Dockerfile.dev`, `docker-compose.dev.yml`, and `src/` — matching what's actually deployed, closing the "folder holds two purposes" ambiguity this used to create.

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

See [`deploy-api.md`](deploy-api.md) — ddashboard can trigger this same script via an authenticated HTTP API instead of SSHing in manually.

---

## New .NET-dev domain — full checklist

Reference run: `osgar.datahub.foxcraft.digital` (Blazor Server, .NET 10). Complements the patterns above and the Caddy notes in [`infrastructure.md`](infrastructure.md).

1. **App folder + source** under `/opt/apps/dotnet/<subdomain>/src` — clone the repo, or for an empty repo: `dotnet new blazor -n <Name> --interactivity Server`
2. **`Dockerfile.dev` + `docker-compose.dev.yml`** after the `advoapp-dev` pattern above. Adjust the host port if `5080` is already taken (e.g. `127.0.0.1:5081:8080`)
3. **DNS A-Record** via Hetzner Cloud API (see [`infrastructure.md`](infrastructure.md) — zone `567656`)
4. **Caddy block** with explicit Let's Encrypt production `dir` pin:
   ```caddyfile
   <subdomain>.foxcraft.digital {
       tls {
           issuer acme {
               email projects@foxcraft.digital
               dir https://acme-v02.api.letsencrypt.org/directory
               dns hetzner {env.HETZNER_API_TOKEN}
           }
       }
       reverse_proxy <container-name>:8080
   }
   ```
5. **`docker compose restart caddy`** from `/opt/services` (not `up -d` alone). Watch issuance logs; if stuck, diagnose/delete stale `_acme-challenge` TXT per infrastructure Issue 2, then restart again
6. **`docker compose -f docker-compose.dev.yml up -d`** in the app folder
7. **`curl -sI https://<subdomain>`** — `502` for the first seconds/minutes after start is normal while `dotnet watch` restores/builds

Do **not** omit `dir` — without it Caddy can fall back to ZeroSSL or Let's Encrypt staging; staging certs are untrusted and leave the domain effectively broken (details in infrastructure Issue 3).

### Checklist addition — restart policy is not optional

Every service in every `docker-compose*.yml` on this server should have `restart: unless-stopped` unless there's a specific reason not to (e.g. a genuinely one-shot job). This was missed on both `advoapp-dev` and `osgar-datahub-dev` when they were first created — both used the same template, both had the same gap, and the first one went unnoticed for 3 days. Check for this explicitly in step-by-step reviews of any new `.dev.yml`.
