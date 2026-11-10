# Edison — Coding-Agent Server

**Status:** Rebuilt 2026-10-09 on a **CX33** (old CPX32 renamed `edison-old`, then deleted). Host + `cursor-agent` container live. Host API live at `https://api.edison.foxcraft.digital` (health + `/v1` in one `host-api` service behind Caddy on `edison-net`; run options and workspaces v1 live 2026-11-02). Mounts are on demand via host-api; currently mounted: osgar, dominikliss. `edison.dev` on Stella is pointed at it. **UNVERIFIED:** end-to-end chat in the UI. See [Edison app](#edison-app).

**Edison** is a dedicated Hetzner Cloud box for coding agents (Cursor CLI today; OpenCode later as a second backend). It is **not** part of Stella. Same Cloud project as Atlas — Atlas reads power/backup status via the Cloud API (`/monitoring/edison`). Agents run here so Stella’s LLM / RAG load and the agent sandbox do not starve each other.

**Used for:** Cursor CLI sessions against remote staging trees (first target: `osgar.datahub.foxcraft.digital`). The agent process lives on Edison; the files it edits are a live `sshfs` mount of the remote app — no local git checkout.

---

## Why a separate server

Stella already runs resource-heavy inference (Ollama, chat, backups). An agent sandbox that compiles, greps, and holds SSHFS mounts would compete for CPU/RAM and would itself get throttled when Stella is busy. Edison is sized for one or two interactive sessions.

The agent’s access to staging is the same scope as the existing Cursor IDE Remote-SSH workflow — an explicit owner decision. The difference that matters later: once a web app triggers the agent headless, there should be an audit trail, because a person is no longer watching every command.

---

## Naming

Servers in this stack are named after One Piece Vegapunk satellites. Firewalls, IPs, networks, and similar infrastructure objects get **functional** names, not Vegapunk names.

| Name | Role | Meaning |
|------|------|---------|
| **Stella** | AI / services host | existing dedicated box |
| **Atlas** | operations dashboard | existing Laravel app |
| **Edison** | coding-agent host | “Invention” |

Examples of functional names already in use: primary IPv4 **`edison-ip`** (`178.105.203.54`), Docker network `edison-net` (the network is named after the host so containers can share it; container names themselves stay tool-based — `cursor-agent`, later `opencode-agent`).

---

## Architecture

```
MacBook
  │  via york 78.47.218.190 (operator path, 2026-11-07)
  │  Proton 194.126.177.181 as fallback
  │  ssh edison   (root, key-only, TCP/22)
  │  GET https://api.edison.foxcraft.digital/health
  ▼
Edison  — Hetzner Cloud CX33, Falkenstein, edison-ip 178.105.203.54
  │
  ├── Docker network: edison-net
  │     caddy                 (publishes 443; api.edison.foxcraft.digital)
  │     host-api              (no published port; Docker socket)
  │     cursor-agent          (Cursor CLI + sshfs)
  │     (later) web-app       (same network, reach agents by container name)
  │     (later) OpenCode as a second backend via an adapter
  │
  └── sshfs live mounts, on demand via host-api (catalog hosts only):
        /mnt/osgar.datahub.foxcraft.digital      (mounted)
          → edison@osgar.datahub.foxcraft.digital:/home/edison/app  (port 2201)
        /mnt/dominikliss.foxcraft.digital        (mounted 2026-11-02)
          → edison@dominikliss.foxcraft.digital:/home/edison/app  (port 2204)
                ▼
Stella  — per-app SSH containers (not a host user)
          also Atlas dev + edison.dev + hypershape
          signed POST /system/health → host-api
          edison.dev → POST /v1/runs (bearer)

Atlas prod (2.31.42.13) — firewall + Caddy allow signed /system/health
```

`sshfs` is a FUSE filesystem over SSH. Every read/write on the mount goes to the remote host in real time. There is no second working copy on Edison. `grep`, `find`, `cat`, and `cursor-agent` treat the mount like a normal local directory.

Cursor CLI has **no** native Remote-SSH (the IDE installs a server component on the remote host; the CLI is a single local process). The mount is the workaround that gives the CLI the same “agent sees remote files” result.

host-api owns the catalog and creates mounts with `POST /v1/workspaces/ensure`. See [host-api.md](host-api.md).

---

## Decision (2026-10-09)

Stay with Cursor CLI for now (rules, skills, MCP and login already work). Build the future web app with an **adapter layer** (start task, stream events, abort, resume) so OpenCode (`opencode serve`: HTTP API, SSE, `OPENCODE_SERVER_PASSWORD`) can be added later as a second backend. OpenCode does not read `.cursor/rules` natively (`AGENTS.md` or the opencode-rules plugin would be needed); skills (`SKILL.md`) are portable. Cursor CLI headless: `-p --output-format stream-json`, `--resume <id>`.

---

## Edison app

Own Laravel app (`foxcraftdigital/edison`), separate from Atlas. Sign-in is Atlas OIDC (authorization-code + PKCE; same password + TOTP; Edison keeps its own session and must not accept Atlas API bearers). Atlas side is in [`../atlas/auth.md`](../atlas/auth.md). Edison client wiring is still to do in the Edison repo. Dev instance at `https://edison.dev.foxcraft.digital` on **Stella** (SSH port 2207). Prod (`edison.foxcraft.digital`) is planned, not built. The headless API is live at `api.edison.foxcraft.digital`. When `EDISON_HOST_API_URL` is set, the app calls `POST /v1/runs` (not stubbed). The Laravel workspaces contract (catalog → ensure → repos → run on a leaf) is not built yet. **UNVERIFIED:** end-to-end chat in the UI (including resume).

- App, repo, env, build: [app.md](app.md)
- Dev instance on Stella: [dev-instance.md](dev-instance.md)
- Host API the web app calls: [host-api.md](host-api.md)
- Run options design (modes / models): [run-options.md](run-options.md)

---

## Documents in this folder

| Document | Contents |
|----------|----------|
| [infrastructure.md](infrastructure.md) | Server specs, Hetzner firewall (443 since 2026-11-01), SSH onto Edison, Docker host + `/opt/services` + `edison-net` |
| [cursor-agent.md](cursor-agent.md) | Image, `run.sh`, systemd, sshfs, headless usage, shared-tree warning |
| [app.md](app.md) | Edison Laravel app: repo, origin, included parts, design, env, build checks, plans |
| [host-api.md](host-api.md) | Live host API: health, workspaces v1, `/v1/runs` / abort / SSE / capabilities, `/v1/skills` (2026-11-03) |
| [run-options.md](run-options.md) | Design: modes, models, settings. Phase 1 host live 2026-11-02; Laravel + UI in the Edison repo, unverified on `edison.dev` |
| [skills.md](skills.md) | Global Cursor skills from `foxcraftdigital/dev-docs`. Host sync + `/v1/skills` live 2026-11-03. |
| [dev-instance.md](dev-instance.md) | `edison.dev.foxcraft.digital` on Stella: DNS, Caddy, compose, SSH 2207, `.env` |

**Related Stella-side docs** (the remote the agent mounts):

- Per-app SSH containers, `edison` user (login verified 2026-11-02 on 2201, 2204, 2205, 2206, 2207; rejected on 2202): [`../stella-server/dev-ssh-access.md`](../stella-server/dev-ssh-access.md)
- osgar-datahub permissions / supervisord: [`../stella-server/apps/osgar-datahub/setup.md`](../stella-server/apps/osgar-datahub/setup.md)
- Stella host + `DOCKER-USER` (`edison-ip` reaches 2201–2202 and 2204–2207 from Edison, 2026-11-02): [`../stella-server/infrastructure.md`](../stella-server/infrastructure.md)

---

## Open next steps

Tracked in [`../open-gaps.md`](../open-gaps.md) (Edison section). Short list:

1. Atlas-side `CheckEdisonHealth` / `EdisonHealthRecorder` — not built. See [`../atlas/monitoring.md`](../atlas/monitoring.md)
2. **UNVERIFIED:** end-to-end chat in the `edison.dev` UI (including resume via the app)
3. Laravel: switch to `GET /v1/workspaces` → `ensure` → `repos` → run on the leaf; key chats by `host + slug`
4. advoapp.finditoo (2202): no `edison` user or key — `ensure` returns 422
5. Full reboot test of the Edison box (only `systemctl restart cursor-agent` was tested)
6. **UNVERIFIED:** Atlas Edison card extra mounts from `mounts.json`
7. First-turn abort — emit an early `session` SSE event so the first turn can be stopped
8. **TODO (Dominik):** design a separate git worktree or clone — runs use `--force` on the live staging tree
9. **TODO (Dominik):** confirm Stella on Edison TCP/22 and ICMP is intentional
10. **UNVERIFIED:** signed `POST /system/health` from Atlas prod (`2.31.42.13`) with the atlas-prod key
11. **TODO (Dominik):** API key for unattended runs (`CURSOR_API_KEY`) — not set up yet
12. Bump the `stella-docs` submodule pointer in the Edison and Atlas repos
13. **TODO (Dominik):** record the `dev-docs` deploy-key fingerprint in [skills.md](skills.md)
14. **`git-push-with-tag` is global on live staging trees** — decide allowlist vs IDE-only. See [skills.md](skills.md)
15. york access gateway (2026-11-07) — [`../york/README.md`](../york/README.md); open whitelist leftovers in [`../open-gaps.md`](../open-gaps.md)
