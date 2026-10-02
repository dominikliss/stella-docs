# Edison — Coding-Agent Server

**Status:** Host + first agent container live as of 2026-09-22. Web UI for headless runs is not built yet.

**Edison** is a dedicated Hetzner Cloud box for coding agents (Cursor CLI today; OpenCode / Claude Code later). It is **not** part of Stella. Agents run here so Stella’s LLM / RAG load and the agent sandbox do not starve each other.

**Used for:** interactive Cursor CLI sessions against remote staging trees (first target: `osgar.datahub.foxcraft.digital`). The agent process lives on Edison; the files it edits are a live `sshfs` mount of the remote app — no local git checkout.

---

## Why a separate server

Stella already runs resource-heavy inference (Ollama, chat, backups). An agent sandbox that compiles, greps, and holds SSHFS mounts would compete for CPU/RAM and would itself get throttled when Stella is busy. Edison is sized for one or two interactive sessions and can be resized independently.

The agent’s access to staging is the same scope as the existing Cursor IDE Remote-SSH workflow — an explicit owner decision. The difference that matters later: once a web app triggers the agent headless, there should be an audit trail, because a person is no longer watching every command.

---

## Naming

Servers in this stack are named after One Piece Vegapunk satellites. Firewalls, IPs, networks, and similar infrastructure objects get **functional** names, not Vegapunk names.

| Name | Role | Meaning |
|------|------|---------|
| **Stella** | AI / services host | existing dedicated box |
| **Atlas** | operations dashboard | existing Laravel app |
| **Edison** | coding-agent host | “Invention” |

Examples of functional names already in use: primary IP `dev-agent-ip`, Docker network `edison-net` (the network is named after the host so containers can share it; container names themselves stay tool-based — `cursor-agent`, later `opencode-agent`).

---

## Architecture

```
MacBook (trusted IP 194.126.177.181)
  │  ssh edison   (root, key-only, TCP/22)
  ▼
Edison  — Hetzner Cloud CPX32, Falkenstein
  │
  ├── Docker network: edison-net
  │     cursor-agent          (Cursor CLI + sshfs)
  │     (later) web-app       (same network, reach agents by container name)
  │     (later) opencode-agent / claude-code-agent
  │
  └── sshfs live mount, not a copy:
        /mnt/osgar.datahub.foxcraft.digital
          → edison@osgar.datahub.foxcraft.digital:/home/edison/app  (port 2201)
                ▼
Stella  — osgar-datahub-ssh container (per-app SSH, not a host user)
```

`sshfs` is a FUSE filesystem over SSH. Every read/write on the mount goes to the remote host in real time. There is no second working copy on Edison. `grep`, `find`, `cat`, and `cursor-agent` treat the mount like a normal local directory.

Cursor CLI has **no** native Remote-SSH (the IDE installs a server component on the remote host; the CLI is a single local process). The mount is the workaround that gives the CLI the same “agent sees remote files” result.

---

## Documents in this folder

| Document | Contents |
|----------|----------|
| [infrastructure.md](infrastructure.md) | Server specs, Hetzner firewall, SSH onto Edison, Docker host + `edison-net` |
| [cursor-agent.md](cursor-agent.md) | Image, run flags, agent SSH key, `sshfs` mounts, Cursor CLI auth + allowlist |

**Related Stella-side docs** (the remote the agent mounts):

- Per-app SSH containers, port 2201, `edison` user: [`../stella-server/dev-ssh-access.md`](../stella-server/dev-ssh-access.md)
- osgar-datahub permissions / supervisord: [`../stella-server/osgar-datahub-dev-setup.md`](../stella-server/osgar-datahub-dev-setup.md)
- Stella host + `DOCKER-USER` (Edison’s public IP must be allowed on 2201): [`../stella-server/infrastructure.md`](../stella-server/infrastructure.md)

---

## Open next steps

Tracked in [`../open-gaps.md`](../open-gaps.md) (Edison section). Short list:

1. Web app to spawn `cursor-agent` headless over `edison-net`
2. Persist `/root/.ssh/config` (and later mounts) as host volumes — today config dies with the container
3. Further agent images (`opencode-agent`, `claude-code-agent`) on the same network and mount scheme
4. Confirm Stella `DOCKER-USER` ACCEPT for Edison’s public IP on port 2201
5. Open HTTP/HTTPS on Edison’s firewall only when the web app is exposed, still IP-restricted
