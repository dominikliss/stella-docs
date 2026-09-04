# stella-docs

**Single source of truth** for system documentation: **ddashboard** (WordPress theme) and **Stella** (AI / services server), plus how they integrate.

This repository is consumed as a **Git submodule** at `docs/stella-docs` inside the `stella-ddashboard` theme repo. Edit docs **here**, then bump the submodule pointer in the theme.

---

## Layout

| Folder                                     | Contents                                                                                                                        |
| ------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------- |
| [**atlas/**](atlas/)                       | Atlas operations platform: deployments (SSH rsync + Stella deploy API), monitoring (Stella health + WP Sites uptime), tools, settings, .NET → Azure |
| [**stella-server/**](stella-server/)       | Stella machine: infrastructure, Docker, Caddy, **Stella API** (FastAPI), **IMAP sync**, **health-api**, Ollama, per-app SSH |
| [**stella-dashboard/**](stella-dashboard/) | WordPress theme: capabilities, architecture, mail, TrackingTime, design system, **OpenAPI + DB reference** ([`reference/`](stella-dashboard/reference/)), **exported Cursor rules** (`cursor-rules/*.md`) |
| [**integration/**](integration/)           | Cross-cutting: **ddashboard ↔ Stella**, email indexing pipeline                                                                |

---

## Start here

| Document                                                                                   | Description                                                        |
| ------------------------------------------------------------------------------------------ | ------------------------------------------------------------------ |
| [atlas/README.md](atlas/README.md)                                                         | Atlas platform — overview, modules, API reference                  |
| [atlas/deploy-pipeline.md](atlas/deploy-pipeline.md)                                       | Deploy flow: ssh_rsync and stella_deploy_api paths                 |
| [atlas/monitoring.md](atlas/monitoring.md)                                                 | Stella health monitoring + WP Sites uptime (Atlas-side)            |
| [integration/ddashboard-and-stella-server.md](integration/ddashboard-and-stella-server.md) | How both systems connect: network, queues, AI split, security, ops |
| [integration/email-indexing.md](integration/email-indexing.md)                             | Mail v3 → Stella **`emails_v3`** / ChromaDB (contract + WordPress backlog) |
| [stella-server/infrastructure.md](stella-server/infrastructure.md)                         | Servers, ports, Docker, Ollama, Caddy                              |
| [stella-server/dev-ssh-access.md](stella-server/dev-ssh-access.md)                         | Per-app SSH containers for Cursor Remote-SSH (not host users)      |
| [stella-server/osgar-datahub-dev-setup.md](stella-server/osgar-datahub-dev-setup.md)       | osgar-datahub dev env: permissions (ACL), Node, supervisord, SCSS watcher |
| [stella-server/stella-api.md](stella-server/stella-api.md)                                 | FastAPI routes and behaviour                                       |
| [stella-server/imap-sync-service.md](stella-server/imap-sync-service.md)                   | Express + `imapsync` job API (mailbox copy on Stella)              |
| [stella-server/health-api.md](stella-server/health-api.md)                                 | Signed system-status endpoint for Atlas (containers, TLS, disk)    |
| [stella-server/deploy-api.md](stella-server/deploy-api.md)                                 | SSH-signature deploy trigger API                                   |
| [stella-server/client-ip-access.md](stella-server/client-ip-access.md)                     | Per-subdomain client IP grants (two-layer: DOCKER-USER + Caddy)    |
| [stella-server/dotnet-app-deployment.md](stella-server/dotnet-app-deployment.md)           | .NET app pattern on Stella (dev containers → Azure production)     |
| [stella-dashboard/architecture.md](stella-dashboard/architecture.md)                       | Full theme architecture (CPTs, REST, mail, PM, PDF, …)             |
| [stella-dashboard/CAPABILITIES.md](stella-dashboard/CAPABILITIES.md)                       | Product / module overview                                          |
| [stella-dashboard/reference/README.md](stella-dashboard/reference/README.md)                 | **OpenAPI 3** (`dls/v1`, `api/v1`) + **custom DB tables** overview |

---

## Backlog

| File                         | Description                                                   |
| ---------------------------- | ------------------------------------------------------------- |
| [open-gaps.md](open-gaps.md) | Known gaps and next steps (indexing, infra, deferred roadmap) |

---

## Theme development rule

Agents and humans changing ddashboard or Stella integration code should update the matching markdown under **`stella-dashboard/`**, **`stella-server/`**, or **`integration/`** in the **same change series**. See the theme’s `.cursor/rules/documentation-source-of-truth.mdc`.
