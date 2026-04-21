# Cursor rule export: `documentation-source-of-truth.mdc`

_Source of agent rule in theme repo: `.cursor/rules/documentation-source-of-truth.mdc`._

---

# Documentation — `stella-docs` is the single source of truth

**There is no second set of product docs** in this theme. All narrative and reference documentation for **ddashboard** (WordPress) and **Stella** (AI server) lives in the Git submodule:

| Path in theme | GitHub repo |
|---------------|-------------|
| **`docs/stella-docs/`** | `dominikliss/stella-docs` |

Folder roles inside the submodule:

| Folder | Use for |
|--------|---------|
| **`stella-dashboard/`** | Theme: capabilities, architecture, mail, TrackingTime, **`reference/`** (OpenAPI + DB overview), design docs, `cursor-rules/*.md` exports |
| **`stella-server/`** | Stella host: infrastructure, Docker, Caddy, FastAPI (`stella-api`) |
| **`integration/`** | Cross-cutting: WordPress ↔ Stella, email indexing pipeline |
| **Root** | `README.md`, `open-gaps.md` |

**Allowed in the theme repo:** `docs/README.md` (pointer only), short stubs like root `DESIGN_SYSTEM.md` / `STATS_PAGE_DESIGN_BRIEF.md` that link into `docs/stella-docs/`. **Do not** add full guides under `docs/*.md` or duplicate long architecture text in `.mdc` files — keep `.mdc` for Cursor behaviour; put facts in the submodule.

---

## Mandatory updates during development

When your change affects **how the system works** (not purely internal refactors with zero behaviour change), update the **matching** markdown in **`docs/stella-docs/`** in the **same task** — before you call the work complete.

| If you change… | Update in submodule (at minimum) |
|----------------|----------------------------------|
| REST routes, `dls/v1` / `api/v1` contracts, auth | `stella-dashboard/architecture.md`, `CAPABILITIES.md` if user-visible; **`stella-dashboard/reference/dls-v1.openapi.json`** and/or **`api-v1.openapi.json`** |
| Custom table DDL / new `install-*-tables.php` | `stella-dashboard/reference/database-overview.md` |
| WP options, cron hooks, install/migrations | `architecture.md`, `CAPABILITIES.md`; integration doc if Stella-related |
| Mail/IMAP/spam/client linking | `stella-dashboard/mail-nachrichten.md`, `mail-cron-hosting.md` |
| Email embed / Stella HTTP client / queue | `integration/email-indexing.md`, `integration/ddashboard-and-stella-server.md` |
| Stella server, compose, API behaviour | `stella-server/infrastructure.md`, `stella-server/stella-api.md` |
| ACF fields / REST field shapes | `stella-dashboard/cursor-rules/acf-field-map.md` **and** this repo’s `acf-field-map.mdc` |
| Known gaps / roadmap items | `open-gaps.md` |
| **Substantive** edits to any `.cursor/rules/*.mdc` | Same meaning in **`stella-dashboard/cursor-rules/<name>.md`** (export body + header note) |

If unsure whether docs need a touch: **prefer updating** — operators and future you rely on one tree.

---

## Git workflow (submodule + theme)

1. **`cd docs/stella-docs`** — edit, **`git add`**, **`git commit`**, **`git push`** (when remote exists).
2. **Theme root** — **`git add docs/stella-docs`**, commit **submodule pointer** with your theme changes (or immediately after).

Cloners need: **`git clone --recurse-submodules`** or **`git submodule update --init --recursive`**.

---

## `.mdc` vs submodule markdown

- **`.mdc` here:** Cursor agent rules (short policy, always-apply behaviour). **Not** a duplicate of full architecture.
- **`stella-dashboard/architecture.md`:** Full architecture narrative (CPTs, routes, mail, PDF, …).
- **`cursor-rules/*.md`:** Mirrors of `.mdc` for humans / Notebook LM — **refresh the matching `.md` when you change the matching `.mdc` in a way that would confuse readers**.

---

## Do not

- Reintroduce long-form **`docs/*.md`** in the theme without first merging into `stella-docs`.
- Put **secrets** in `.gitmodules`, markdown, or examples (PATs, raw API keys).
- Declare a feature “done” when behaviour changed but **no** doc path above was reviewed for updates.
