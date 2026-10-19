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
| **Atlas** | Operations platform (deploys, health, WP sites, IMAP copy, TrackingTime import, Trello push, notifications) |
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
| [**atlas/**](atlas/) | Atlas: architecture, production host runbook, host API, `atlas.dev` instance, auth, Atlas Connect, deploys, monitoring, tools, projects / Trello, notifications |
| [**stella-server/**](stella-server/) | Stella host: infra, APIs, per-app SSH; app runbooks under [`apps/`](stella-server/apps/) |
| [**stella-dashboard/**](stella-dashboard/) | ddashboard product docs, OpenAPI, Cursor rule exports |
| [**edison/**](edison/) | Edison host, `edison-net`, Cursor CLI / sshfs |

---

## Branch

**Work on `main` only.** Do not create feature branches, topic branches, or PR branches in this repo. Commit and push straight to `origin/main`.

Hit 2026-10-18: a docs commit was pushed to `docs/dynamic-health-backup-2026-10-10` because that was the local checkout. That branch is gone; `main` is the only branch. Keep it that way.

## Done means

1. Edit the matching folder in this repo. Checkout is `main`, in sync with `origin/main`.
2. Commit and push **`main`** on `dominikliss/stella-docs`. Never `checkout -b` / never push a second branch.
3. Bump the submodule pointer in the parent (theme and/or Atlas).
