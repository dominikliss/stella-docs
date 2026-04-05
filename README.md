# stella-docs

**Single source of truth** for system documentation: **ddashboard** (WordPress theme) and **Stella** (AI / services server), plus how they integrate.

This repository is consumed as a **Git submodule** at `docs/stella-docs` inside the `stella-ddashboard` theme repo. Edit docs **here**, then bump the submodule pointer in the theme.

---

## Layout

| Folder | Contents |
|--------|----------|
| [**stella-server/**](stella-server/) | Stella machine: infrastructure, Docker, Caddy, **Stella API** (FastAPI) |
| [**stella-dashboard/**](stella-dashboard/) | WordPress theme: capabilities, architecture, mail, TrackingTime, design system, **exported Cursor rules** (`cursor-rules/*.md`) |
| [**integration/**](integration/) | Cross-cutting: **ddashboard ↔ Stella**, email indexing pipeline |

---

## Start here

| Document | Description |
|----------|-------------|
| [integration/ddashboard-and-stella-server.md](integration/ddashboard-and-stella-server.md) | How both systems connect: network, queues, AI split, security, ops |
| [integration/email-indexing.md](integration/email-indexing.md) | Email embed queue → ChromaDB via Stella API |
| [stella-server/infrastructure.md](stella-server/infrastructure.md) | Servers, ports, Docker, Ollama, Chroma |
| [stella-server/stella-api.md](stella-server/stella-api.md) | FastAPI routes and behaviour |
| [stella-dashboard/architecture.md](stella-dashboard/architecture.md) | Full theme architecture (CPTs, REST, mail, PM, PDF, …) |
| [stella-dashboard/CAPABILITIES.md](stella-dashboard/CAPABILITIES.md) | Product / module overview |

---

## Backlog

| File | Description |
|------|-------------|
| [open-gaps.md](open-gaps.md) | Known gaps and next steps (indexing, infra, deferred roadmap) |

---

## Theme development rule

Agents and humans changing ddashboard or Stella integration code should update the matching markdown under **`stella-dashboard/`**, **`stella-server/`**, or **`integration/`** in the **same change series**. See the theme’s `.cursor/rules/documentation-source-of-truth.mdc`.
