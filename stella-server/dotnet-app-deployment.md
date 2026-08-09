# .NET App Deployment Pattern

Established 2026-08-06 with `finditoo-advoapp` (`advoapp`) as the first implementation. This is the template for any future .NET app on Stella.

## Three environments, three different mechanisms

| Environment | Where | How | URL |
|---|---|---|---|
| **Dev** | Stella, Docker | `dotnet watch` hot-reload, bind-mounted source | `advoapp.finditoo.foxcraft.digital` (currently — see note below) |
| **Staging** | Stella, Docker | Built image, manual/triggered rebuild | not currently wired to a subdomain — see note below |
| **Production** | Azure App Service | Clean checkout → `dotnet publish` → zip → `az webapp deploy` | Azure-assigned `*.azurewebsites.net` (custom domain not yet configured) |

**⚠️ Current state note:** The original plan was dev + staging as two separate Docker containers, with staging (the built, non-watching container) exposed at `advoapp.finditoo.foxcraft.digital`. In practice, Dominik asked for that public subdomain to show the **live dev container** instead, since "the production URL is completely different" (i.e., Azure is production, so the Stella subdomain only ever needs to reflect current dev work). As of now, the Caddyfile routes `advoapp.finditoo.foxcraft.digital` → `advoapp-dev:8080`. The originally-built plain staging container (`docker-compose.yml`, no hot-reload) still exists and can be rebuilt with `docker compose build && up -d` in its folder, but nothing on the public internet points to it currently.

---

## Folder layout

```
/opt/apps/dotnet/
  ├── advoapp.finditoo.foxcraft.digital/            ← dev + "staging" container definitions
  │     ├── src/                                     ← git clone, live-edited via Cursor/SSH
  │     ├── Dockerfile                                ← staging build (dotnet publish, runtime image)
  │     ├── docker-compose.yml                         ← staging container def (not currently exposed via Caddy)
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

Run manually (start it, view logs): `docker compose -f docker-compose.dev.yml up`

---

## Staging container (built, not currently exposed)

`Dockerfile`:
```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
WORKDIR /src
COPY src/ .
RUN dotnet restore
RUN dotnet publish -c Release -o /app

FROM mcr.microsoft.com/dotnet/aspnet:10.0 AS runtime
WORKDIR /app
COPY --from=build /app .
EXPOSE 8080
ENTRYPOINT ["dotnet", "finditoo.advoapp.dll"]
```

`docker-compose.yml`:
```yaml
services:
  advoapp.finditoo.foxcraft.digital:
    build: .
    container_name: advoapp.finditoo.foxcraft.digital
    environment:
      - ASPNETCORE_URLS=http://+:8080
      - ASPNETCORE_ENVIRONMENT=Development
    networks:
      - edge
    restart: unless-stopped

networks:
  edge:
    external: true
```

Rebuild manually: `docker compose build && docker compose up -d` from this folder.

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
