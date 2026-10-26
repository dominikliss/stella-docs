# Edison app (Laravel)

Web app for Edison. It is its own Laravel app, separate from Atlas; an Atlas feature was considered and rejected. Auth is the same as Atlas: ddashboard token login.

**Dev instance:** [`dev-instance.md`](dev-instance.md)  
**Edison host (coding-agent server):** [`README.md`](README.md)

Status 2026-10-26: foundation only (auth, roles, shell, design system, empty Dashboard, generic Settings). The agent backend is stubbed: there is no link to the real `cursor-agent` on Edison yet.

---

## Repo

| | |
|---|---|
| **Repo** | `git@github.com:foxcraftdigital/edison.git`, branch `main` |
| **Commit 1** | `bca0424` "Laravel skeleton", pushed 2026-10-26 |
| **Commit 2** | `acb37cc` (auth, design system, shell, submodules, agent context), pushed 2026-10-26 with tag **`0.1.1`**. The version is shown in the login page footer. |
| **Stack** | Laravel 13 on PHP 8.5, Spatie Permission, Pest 4, React 18 + TypeScript + Vite + TanStack Router / Query, Tailwind v4 |

### Origin

A fresh Laravel skeleton plus parts copied by Cursor from a read-only clone of the Atlas repo (`~/atlas-ref`, outside the app tree). **TODO (Dominik):** delete `~/atlas-ref` in the `-ssh` container (it is also lost on any rebuild, see [`dev-instance.md`](dev-instance.md)).

Not carried over:

- Atlas history
- Atlas features (deployments, WP sites, monitoring, tools, projects, notifications, Hetzner / GitHub / Brevo settings)
- Atlas DB data
- Atlas env values

The `APP_KEY` and the database are fresh.

---

## What is included

- **ddashboard token auth:** `app/Domain/Auth` with `AllowedIpMiddleware` and the token driver.
  - Tokens are verified with `GET /dls/v1/auth/verify`.
  - Login, refresh, and logout are proxied to ddashboard.
- **Roles:** Spatie roles/permissions (`config/permission.php`, `RolesSeeder`).
- **Backend scaffolding:** `Team` model, `EdisonServiceProvider`, `config/edison.php`.
- **Migrations:** teams + users update, permission tables, `prepare_users_for_ddashboard_auth`.
- **Frontend:** React + TypeScript + Vite + TanStack Router (`resources/js`). It contains the app shell, the login page, a generic Settings page, and an empty Dashboard.
- **Tests:** Pest.

### Design

Dark-only "Technical Precision" system, copied from Atlas.

| File | Role |
|------|------|
| `design/DESIGN.md` | Spec |
| `design-tokens/tokens.json` | Tokens |
| `resources/css/app.css` | Tailwind v4 `@theme` block |
| `tailwind.config.ts` | Mirrors the tokens |
| `design/components.html` | Static component reference |

### Agent context

- `CLAUDE.md` and `AGENTS.md`, merged with the Laravel defaults
- `.cursor/`, including a `git-push-with-tag` skill
- `.claude/` skills (Laravel and testing best practices)
- `.mcp.json`, `boost.json`, `config/boost.php` (`laravel/boost`, `laravel/mcp`)

The only "atlas" mentions left in the repo are intentional: the design language, the auth contract link, and "do not port Atlas features".

---

## Submodules

Same as Atlas, SSH URLs so the forwarded agent works:

| Path | Remote |
|------|--------|
| `docs/stella-docs` | `git@github.com:dominikliss/stella-docs.git` |
| `docs/dev-docs` | `git@github.com:foxcraftdigital/dev-docs.git` |

Submodules pin a commit. After changes in stella-docs, bump the pointer in the Edison repo. Collaborators on the Edison repo can read stella-docs (infrastructure details) through the submodule if they have access to that repo.

---

## Env

Edison-specific variables use the `EDISON_` prefix (`config/edison.php`). All are documented in `.env.example`.

| Key | Notes |
|-----|-------|
| `EDISON_DDASHBOARD_URL` | `config('edison.ddashboard_url')`, empty by default. **Must be set.** |
| `EDISON_ALLOWED_IPS` | App-level IP allowlist; empty disables it. `/api/health` is always reachable. |
| `EDISON_DEMO_MODE` | `true` swaps in a fake auth driver (no ddashboard calls). |

### Incident 2026-10-26 — empty `EDISON_DDASHBOARD_URL`

With the variable empty, the login call failed with `Illuminate\Http\Client\ConnectionException` "URI must include a scheme and host" (log lines at 17:52 and 18:05; details in [`dev-instance.md`](dev-instance.md#incident-2026-10-26--empty-edison_ddashboard_url)). Resolved by setting the variable. **TODO (Dominik):** make the driver throw a clear "EDISON_DDASHBOARD_URL is not set" error.

---

## Build and checks (2026-10-26)

- `composer install`
- `npm install` (0 vulnerabilities)
- `npm run build` (`tsc && vite build`, 214 modules, no errors)
- `php artisan migrate` (nothing to migrate; the migrations had already run)
- Pest test suite green before commit 2
- Secrets check before commit:
  - No tokens, keys, or Bearer strings in `.mcp.json`, `.env.testing`, `boost.json`, `.claude`, `.cursor`.
  - `.env` is gitignored.

**UNVERIFIED:**

- End-to-end login through the real ddashboard. During setup, login / me / refresh / logout were only exercised against a local mock ddashboard.
- Whether ddashboard's IP allowlist accepts Stella (`95.217.144.93`) for `auth/verify`. The later prod instance would also need `178.105.203.54` (`edison-ip`). The allowlist is probably an `.htaccess` under `public_html` on the ddashboard host (see [`../open-gaps.md`](../open-gaps.md)).

---

## Planned (not built)

- **Prod:** `edison.foxcraft.digital` on the Edison server.
- **Headless API:** `api.edison.foxcraft.digital`, same pattern as the Atlas host API ([`../atlas/host-api.md`](../atlas/host-api.md)), so Atlas can use Edison. Not started.
- **Atlas-side client:** flag-gated, like self-deploy.
