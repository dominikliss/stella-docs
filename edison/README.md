# Edison — Coding-Agent Server

**Status:** Rebuilt 2026-10-09 on a **CX33** (old CPX32 renamed `edison-old`, then deleted). Host + `cursor-agent` container live. Web UI for headless runs is not built yet.

**Edison** is a dedicated Hetzner Cloud box for coding agents (Cursor CLI today; OpenCode later as a second backend). It is **not** part of Stella. Agents run here so Stella’s LLM / RAG load and the agent sandbox do not starve each other.

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
MacBook (trusted IP 194.126.177.181)
  │  ssh edison   (root, key-only, TCP/22)
  ▼
Edison  — Hetzner Cloud CX33, Falkenstein, edison-ip 178.105.203.54
  │
  ├── Docker network: edison-net
  │     cursor-agent          (Cursor CLI + sshfs)
  │     (later) web-app       (same network, reach agents by container name)
  │     (later) OpenCode as a second backend via an adapter
  │
  └── sshfs live mount, not a copy:
        /mnt/osgar.datahub.foxcraft.digital
          → edison@osgar.datahub.foxcraft.digital:/home/edison/app  (port 2201)
                ▼
Stella  — osgar-datahub-ssh container (per-app SSH, not a host user)
```

`sshfs` is a FUSE filesystem over SSH. Every read/write on the mount goes to the remote host in real time. There is no second working copy on Edison. `grep`, `find`, `cat`, and `cursor-agent` treat the mount like a normal local directory.

Cursor CLI has **no** native Remote-SSH (the IDE installs a server component on the remote host; the CLI is a single local process). The mount is the workaround that gives the CLI the same “agent sees remote files” result.

`dominikliss.foxcraft.digital` has an SSH config entry (port 2204) but **no sshfs mount yet**.

---

## Decision (2026-10-09)

Stay with Cursor CLI for now (rules, skills, MCP and login already work). Build the future web app with an **adapter layer** (start task, stream events, abort, resume) so OpenCode (`opencode serve`: HTTP API, SSE, `OPENCODE_SERVER_PASSWORD`) can be added later as a second backend. OpenCode does not read `.cursor/rules` natively (`AGENTS.md` or the opencode-rules plugin would be needed); skills (`SKILL.md`) are portable. Cursor CLI headless: `-p --output-format stream-json`, `--resume <id>`.

---

## Documents in this folder

| Document | Contents |
|----------|----------|
| [infrastructure.md](infrastructure.md) | Server specs, Hetzner firewall, SSH onto Edison, Docker host + `edison-net` |
| [cursor-agent.md](cursor-agent.md) | Image, `run.sh`, systemd, sshfs, headless usage, shared-tree warning |

**Related Stella-side docs** (the remote the agent mounts):

- Per-app SSH containers, ports 2201 / 2204, `edison` user: [`../stella-server/dev-ssh-access.md`](../stella-server/dev-ssh-access.md)
- osgar-datahub permissions / supervisord: [`../stella-server/apps/osgar-datahub/setup.md`](../stella-server/apps/osgar-datahub/setup.md)
- Stella host + `DOCKER-USER` (Edison `edison-ip` `178.105.203.54` is ACCEPTed on 2201 and 2204): [`../stella-server/infrastructure.md`](../stella-server/infrastructure.md)

---

## Open next steps

Tracked in [`../open-gaps.md`](../open-gaps.md) (Edison section). Short list:

1. Web app with an adapter layer (start / stream / abort / resume) over `edison-net`
2. **TODO (Dominik):** design a separate git worktree or clone — the agent must not `checkout` / `switch` / `stash` / `reset` / `commit` on the live osgar staging tree
3. **TODO (Dominik):** API key for unattended runs (`CURSOR_API_KEY`) — not set up yet
4. **TODO (Dominik):** default model — check `cursor-agent models` / `--help`, set `--model` explicitly
5. sshfs mount for `dominikliss.foxcraft.digital` (SSH config exists; no mount yet)
6. Open HTTP/HTTPS on Edison’s firewall only when the web app is exposed, still IP-restricted
