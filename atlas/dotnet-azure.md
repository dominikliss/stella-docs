# .NET → Azure App Service via Atlas

Atlas's native deploy pipeline (git clone → rsync) doesn't cover compiled apps — .NET requires `dotnet publish` before any files are deployable. The solution is to use Atlas's **post-deploy endpoint** to trigger Stella's [deploy-api](../stella-server/deploy-api.md), which handles the build and Azure push.

---

## Architecture

```
Atlas triggers run
  └── rsync is skipped (deploy config has no SSH destinations)
        └── post-deploy endpoint → stella-deployment-api.foxcraft.digital
              └── deploy-advoapp.sh on Stella
                    ├── git pull (clean checkout of master)
                    ├── docker run dotnet publish
                    ├── zip publish output
                    └── az webapp deploy → Azure App Service
```

**Why not use Atlas's SSH rsync at all:** Azure App Service is not an SSH server you can rsync files into. The build (`dotnet publish`) must run on a machine with the .NET SDK — Stella handles this via a Docker container, keeping the SDK off the host OS.

---

## Setup

### 1. Deploy config in Atlas

Create a `DeployConfig` with:

| Field | Value |
|---|---|
| `github_repo` | `foxcraftdigital/finditoo-advoapp` (or the relevant repo) |
| `branch` | `master` |
| `environment` | `production` |
| `destinations` | **none** — leave the destinations list empty |

The config's purpose here is to serve as a trigger record (tracks run history, commit SHA, status) and to call the post-deploy endpoint. It does not rsync anything itself.

> **Note:** Atlas currently requires at least one destination to run. Until this is relaxed, create a placeholder destination pointing at Stella's own SSH server with a no-op path, or use the deploy-api directly without Atlas. See the open-gaps note below.

### 2. Post-deploy endpoint

In the destination's **Post-Deploy Process URL**, set:

```
https://stella-deployment-api.foxcraft.digital/deploy/advoapp-production
```

Atlas will POST to this URL after the (empty) rsync completes. The deploy-api receives the call, verifies the SSH signature, and runs `deploy-advoapp.sh`.

**Auth:** the post-deploy endpoint uses SSH-signature authentication — see [deploy-api.md](../stella-server/deploy-api.md#auth-mechanism). Atlas needs to sign the payload with its SSH key, which must be listed in Stella's `allowed_signers` file.

> As of 2026-08-06, this wiring is not yet implemented in Atlas — Atlas calls post-deploy endpoints as plain unauthenticated HTTP POSTs. Until Atlas supports SSH-signed requests, trigger the deploy-api directly (curl from ddashboard, or manually) rather than via Atlas's post-deploy hook.

---

## Current state (2026-08-06)

The .NET → Azure deploy works end-to-end but **not yet via Atlas**. The current trigger path is:

1. **Manual:** SSH to Stella, run `/opt/services/azure-deploy/deploy-advoapp.sh`
2. **From ddashboard:** POST to `stella-deployment-api.foxcraft.digital` with a signed payload (see [deploy-api.md](../stella-server/deploy-api.md))

Atlas integration is planned — see open gaps below.

---

## Planned Atlas integration

Two options, in order of implementation complexity:

### Option A — Atlas triggers deploy-api via post-deploy hook (medium)

Extend Atlas's `PostDeployEndpointService` to support an SSH-signed request mode. When a destination's post-deploy URL is on an allowlisted domain (e.g. `*.foxcraft.digital`), Atlas signs the payload with its SSH key before POSTing, matching the deploy-api's expected format.

**What changes in Atlas:**
- `PostDeployEndpointService`: detect signed-request mode, call `ssh-keygen -Y sign`, attach payload + signature to request body
- Atlas's SSH public key added to Stella's `deploy-api/allowed_signers`
- New deploy config with a placeholder destination + post-deploy endpoint URL

### Option B — New `remote_command` destination type (larger)

Add a deploy destination type that, instead of rsync, SSHes into a server and runs a configured shell command. For .NET, that command would be `/opt/services/azure-deploy/deploy-advoapp.sh`.

**What changes in Atlas:**
- New `type` column on `DeployDestination` (`rsync` | `remote_command`)
- New `remote_command` field on the destination
- `DeployRunJob`: branch on type, call new `RemoteCommandDeployService` instead of rsync
- Frontend: destination form shows command field when type = `remote_command`

This is more general — any post-deploy script on any SSH server becomes triggerable from Atlas, not just the deploy-api pattern.

---

## Open gap

Atlas's `POST /api/deploy/runs` currently requires at least one destination with a valid server and path. A config with zero destinations (pure post-deploy-endpoint trigger) is not yet supported. This is tracked in `open-gaps.md`.
