# stella-docs

**Single source of truth** for foxcraft.digital: **ddashboard**, **Stella**, **Atlas**, and **Edison**.

Vendored as `docs/stella-docs` in the theme and in Atlas. Edit **here**, push this repo, then bump the parent submodule pointer.

---

## Naming

Servers use One Piece Vegapunk-satellite names. Firewalls, IPs, and networks use **functional** names.

| Name | Role |
|------|------|
| **ddashboard** | WordPress business app (CRM, accounting, mail, PM, AI chat) |
| **Stella** | AI / services host (dedicated) |
| **Atlas** | Operations platform (deploys, health, WP sites, IMAP copy) |
| **Edison** | Coding-agent host (Hetzner Cloud) — “Invention” |

---

## Start here

| Document | Description |
|----------|-------------|
| [integration/system-overview.md](integration/system-overview.md) | **The company map** — who talks to whom |
| [atlas/README.md](atlas/README.md) | Atlas ops platform |
| [stella-server/README.md](stella-server/README.md) | Stella host and services |
| [stella-dashboard/README.md](stella-dashboard/README.md) | ddashboard theme (folder name is historical) |
| [edison/README.md](edison/README.md) | Edison coding-agent host |
| [open-gaps.md](open-gaps.md) | Known gaps and backlog |

Topic pages live in each system README. Shared contracts stay as two files (Atlas half + Stella half) — update both in the same commit.

---

## Layout

| Folder | Contents |
|--------|----------|
| [**integration/**](integration/) | System map, ddashboard ↔ Stella AI |
| [**atlas/**](atlas/) | Atlas: architecture, auth, Atlas Connect, deploys, monitoring, tools |
| [**stella-server/**](stella-server/) | Stella host: infra, APIs, per-app SSH; app runbooks under [`apps/`](stella-server/apps/) |
| [**stella-dashboard/**](stella-dashboard/) | ddashboard product docs, OpenAPI, Cursor rule exports |
| [**edison/**](edison/) | Edison host, `edison-net`, Cursor CLI / sshfs |

---

## Done means

1. Edit the matching folder in this repo.
2. Commit and push `dominikliss/stella-docs`.
3. Bump the submodule pointer in the parent (theme and/or Atlas).
