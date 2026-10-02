# stella-docs

**Single source of truth** for system documentation: **ddashboard** (WordPress theme), **Stella** (AI / services server), **Atlas** (ops platform), and **Edison** (coding-agent host), plus how they integrate.

This repository is consumed as a **Git submodule** at `docs/stella-docs` inside the `stella-ddashboard` theme repo. Edit docs **here**, then bump the submodule pointer in the theme.

---

## Naming

Servers use One Piece Vegapunk-satellite names. Firewalls, IPs, networks, and similar objects use **functional** names.

| Name | Role |
|------|------|
| **ddashboard** | WordPress business app (CRM, accounting, mail, PM, AI chat) |
| **Stella** | AI / services host (dedicated) |
| **Atlas** | Operations platform (deploys, health, WP sites, IMAP copy) |
| **Edison** | Coding-agent host (Hetzner Cloud) — “Invention” |

---

## Layout

| Folder                                     | Contents                                                                                                                        |
| ------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------- |
| [**atlas/**](atlas/)                       | Atlas operations platform: architecture, auth, Atlas Connect, deployments, monitoring, IMAP tools, .NET → Azure |
| [**edison/**](edison/)                     | Edison coding-agent server: host, Hetzner firewall, Docker `edison-net`, Cursor CLI container, sshfs remote mounts |
| [**stella-server/**](stella-server/)       | Stella machine: infrastructure, Docker, Caddy, **Stella API** (FastAPI), **IMAP sync**, **health-api**, Ollama, per-app SSH |
| [**stella-dashboard/**](stella-dashboard/) | WordPress theme: capabilities, architecture, mail, TrackingTime, design system, **OpenAPI + DB reference** ([`reference/`](stella-dashboard/reference/)), **exported Cursor rules** (`cursor-rules/*.md`) |
| [**integration/**](integration/)           | Cross-cutting: **full system map**, ddashboard ↔ Stella AI, email indexing pipeline |

---

## Start here

| Document                                                                                   | Description                                                        |
| ------------------------------------------------------------------------------------------ | ------------------------------------------------------------------ |
| [integration/system-overview.md](integration/system-overview.md)                           | **Start here for the whole company stack** — Atlas, ddashboard, Stella, Edison |
| [atlas/README.md](atlas/README.md)                                                         | Atlas platform — overview, modules, API reference                  |
| [atlas/architecture.md](atlas/architecture.md)                                             | Atlas code layout, hosting, queue, SPA routes                      |
| [atlas/auth.md](atlas/auth.md)                                                             | Atlas login via ddashboard tokens                                  |
| [atlas/atlas-connect/](atlas/atlas-connect/)                                               | WP plugin: one-click admin login + encrypted S3 backups            |
| [atlas/deploy-pipeline.md](atlas/deploy-pipeline.md)                                       | Deploy flow: ssh_rsync and stella_deploy_api paths                 |
| [atlas/monitoring.md](atlas/monitoring.md)                                                 | Stella health monitoring + WP Sites uptime (Atlas-side)            |
| [atlas/tools.md](atlas/tools.md)                                                           | IMAP mailbox copy via Stella imap-sync                             |
| [edison/README.md](edison/README.md)                                                       | Edison — dedicated coding-agent host (why separate, naming, sshfs) |
| [edison/infrastructure.md](edison/infrastructure.md)                                       | Edison server, Hetzner firewall, SSH, Docker `edison-net`          |
| [edison/cursor-agent.md](edison/cursor-agent.md)                                           | Cursor CLI container, agent key, sshfs mounts, allowlist           |
| [integration/ddashboard-and-stella-server.md](integration/ddashboard-and-stella-server.md) | ddashboard ↔ Stella **AI chat** path (not Atlas)                   |
| [integration/email-indexing.md](integration/email-indexing.md)                             | Mail v3 → Stella **`emails_v3`** / ChromaDB (decommissioned)       |
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

Agents and humans changing ddashboard, Stella, Atlas, or Edison should update the matching markdown under **`stella-dashboard/`**, **`stella-server/`**, **`atlas/`**, **`edison/`**, or **`integration/`** in the **same change series**. See the theme’s `.cursor/rules/documentation-source-of-truth.mdc`.
