# .NET → Azure App Service via Atlas

Compiled .NET apps cannot use Atlas `ssh_rsync` (no `dotnet publish` on the Atlas host; Azure App Service is not an rsync target). The live path is Atlas deploy type **`stella_deploy_api`**: Atlas signs a trigger, Stella builds and pushes.

Stella-side scripts and Azure: [`../stella-server/dotnet-app-deployment.md`](../stella-server/dotnet-app-deployment.md), [`../stella-server/deploy-api.md`](../stella-server/deploy-api.md).  
Atlas job steps: [`deploy-pipeline.md`](deploy-pipeline.md#stella_deploy_api-path).

---

## Live architecture (since 2026-08-07)

```
Atlas UI / POST /api/deploy/runs
  └── DeployConfig.type = stella_deploy_api
        └── DeployRunJob::handleStellaApiDeploy()
              ├── sign "deploy-{app}:{timestamp}"
              ├── POST {STELLA_DEPLOY_API_URL}/deploy/{app}
              └── poll /deploy/{app}/status/{job_id}
                    └── Stella deploy-advoapp.sh (or app-specific script)
                          ├── git pull
                          ├── docker run dotnet publish
                          ├── zip
                          └── az webapp deploy → Azure App Service
```

Confirmed from Atlas (`dev.atlas.foxcraft.digital`) on 2026-08-07: app list, trigger, poll to completion, commit SHA + message stored on the run. See the verification note in [`../stella-server/deploy-api.md`](../stella-server/deploy-api.md).

---

## Deploy config in Atlas

| Field | Value |
|-------|--------|
| `type` | `stella_deploy_api` |
| `stella_deploy_url` | `https://stella-deployment-api.foxcraft.digital` (overridden by `STELLA_DEPLOY_API_URL` when set) |
| `stella_deploy_app` | App name registered on Stella (e.g. `advoapp-production`, `advoapp-dev`) |
| `destinations` | None — Stella-type runs create a single synthetic `DeployRunDestination` for the log / `stella_job_id` |

`GET /api/deploy/stella/apps` fills the app dropdown (signed `list-apps:{ts}`).

Auth is the shared Atlas SSH key as principal `ddashboard` — not a bearer token, and **not** the unsigned post-deploy HTTP hook.

---

## What not to use

| Old idea | Status |
|----------|--------|
| Empty `ssh_rsync` config + post-deploy URL to deploy-api | **Do not use.** `PostDeployEndpointService` is still a plain HTTP POST (no SSH signature). deploy-api would reject it. |
| Placeholder SSH destination just to satisfy “one destination” | **Unnecessary.** `stella_deploy_api` does not require destinations. |
| Manual SSH to Stella to run `deploy-advoapp.sh` | Emergency fallback only. |

---

## Dev vs production on Stella

- **Dev / staging .NET** often stays in Docker on Stella (`advoapp-dev`, `osgar-datahub-dev`) and is still triggered through the same deploy-api app names.
- **Production** (Azure App Service) is the `az webapp deploy` branch inside the Stella script.

Atlas does not distinguish those environments beyond which `stella_deploy_app` you select.
