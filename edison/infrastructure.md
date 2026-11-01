# Edison — Infrastructure

**Status:** Rebuilt 2026-10-09 (old CPX32 renamed `edison-old`, then deleted). TCP/443 opened 2026-11-01 for `api.edison.foxcraft.digital` (IP-restricted). Port 80 stays closed.

**Related:** [README.md](README.md) (role + naming), [cursor-agent.md](cursor-agent.md) (agent container), [host-api.md](host-api.md) (Caddy + host-api)

---

## Server

| | |
|---|---|
| **Name** | Edison |
| **Provider** | Hetzner Cloud |
| **Type** | **CX33** — 4 vCPU shared x86, 8 GB RAM, 80 GB SSD, ~€8.49/mo |
| **OS** | Ubuntu 26.04 LTS |
| **Location** | Falkenstein |
| **Primary IP** | Hetzner name **`edison-ip`**, address **`178.105.203.54`** (Falkenstein). Moved from the old server to this one. Stella `DOCKER-USER` ACCEPTs this IP on ports 2201 and 2204 — confirmed 2026-10-09; no Stella firewall change was needed. |
| **Backups** | Hetzner automatic backups **off** on purpose (sandbox; code lives in git). Take a manual snapshot before risky host changes. |

**Why CX33 (2026-10-09):** cost. Old server was a CPX32 (4 vCPU shared, 8 GB RAM, 160 GB SSD). A rescale from CPX32 was not possible because Hetzner never shrinks disks (160 GB → 80 GB), so a new server was created. CX43/CX53 were shown as unavailable in the console when the choice was made, so the upgrade path from CX33 may require another rebuild. Fallback is a CPX type.

Edison must stay off Stella. Do not colocate agent containers on the Stella dedicated box.

---

## Firewall (Hetzner Cloud)

Inbound, public NIC only. Attached to the new server.

| Rule | Source | Notes |
|------|--------|--------|
| TCP/22 (SSH) | `194.126.177.181`, `95.217.144.93` | Live firewall allows Stella as well as the laptop/VPN IP. Docs previously listed only `194.126.177.181`. **TODO (Dominik):** confirm Stella on 22/ICMP is intentional. |
| ICMP | Same two IPs — **not** Any | Ping restricted the same way as SSH |
| TCP/443 | `194.126.177.181`, `95.217.144.93` (Stella: Atlas dev, edison.dev), `2.31.42.13` (Atlas prod) | Opened 2026-11-01 for `api.edison.foxcraft.digital`. IP-restricted. Port 80 not opened. |

**Verified 2026-10-09** from Stella (`95.217.144.93`): port 22 on `178.105.203.54` was blocked. That check predates the Stella 22/ICMP rule now present in the live firewall.

This firewall does **not** see Docker-internal traffic. Containers on `edison-net` (`caddy` → `host-api`; `host-api` `docker exec`s into `cursor-agent`) talk to each other by container name and never hit these rules. The web app lives on Stella and reaches Caddy on 443.

---

## SSH onto Edison (the host)

- User: `root`
- Auth: key-only. Root `authorized_keys` holds Dominik's MacBook key.
- `/etc/ssh/sshd_config.d/00-no-password.conf` contains `PasswordAuthentication no`; `PermitRootLogin prohibit-password`.
- Verified 2026-10-09: key login works, password login is refused.

Agent key (Edison → Stella remotes, not used for host login): `~/.ssh/edison_agent`, comment `edison-agent`. See [cursor-agent.md](cursor-agent.md).

Local shortcut on the MacBook (`~/.ssh/config`):

```
Host edison
    HostName 178.105.203.54
    User root
    IdentityFile ~/.ssh/id_ed25519
```

Then: `ssh edison`.

---

## Docker on the host

Docker **29.9.0**, Docker Compose **v5.6.0**, installed with the official script (not distro packages):

```bash
curl -fsSL https://get.docker.com | sh
```

User-defined bridge so every agent and the control plane share Docker DNS:

```bash
docker network create edison-net
```

All agent and control-plane containers join `--network edison-net`. Edison appears in the **network** name only. Container and host-folder names follow the **tool**: `cursor-agent`, `~/cursor-agent/`, later `opencode-agent` / `~/opencode-agent/`.

Control plane (same `/opt/services` pattern as Stella), added 2026-11-01:

```
/opt/services/
├── docker-compose.yml    # caddy + host-api, network edison-net (external)
├── caddy/                # publishes 443 only
└── host-api/             # no published port; Docker socket + allowed_signers
```

Named volume `host-api-state` → `/var/lib/host-api` (`sessions.json` only: agent, workspace, timestamps — no prompts).

`ss` on the host shows **22** and **443**. The agent container publishes no ports. `ufw` is inactive — the Hetzner Cloud firewall is the only filter. See [host-api.md](host-api.md).

**Pre-checks 2026-11-01** (before this setup): no API, no Caddy, nothing listening except sshd (22) and systemd-resolved (`127.0.0.53`/`54`). 75 GB disk / 7.6 GB RAM, no swap. `~/cursor-agent` unchanged.

---

## Stella-side access (required for sshfs)

The agent SSHes to Stella’s per-app SSH container, not to Stella’s host SSH:

| | |
|---|---|
| Host | `osgar.datahub.foxcraft.digital` (Stella public IP `95.217.144.93` at setup) |
| Port | **2201** (published by `osgar-datahub-ssh`) |
| User | `edison` — dedicated user inside that container, for revoke-without-touching-devs |

Stella’s `DOCKER-USER` chain ACCEPTs `194.126.177.181`, `23.88.90.12`, and **`178.105.203.54` (`edison-ip`)** on 2201 (and the same three on 2204) and DROPs everything else. Confirmed 2026-10-09 — the IPv4 was moved with the server, so no Stella firewall change was needed.

A second per-app SSH container with an `edison` user exists on port **2204** (`dominikliss.foxcraft.digital`). It is reachable because `edison-ip` is in the `DOCKER-USER` ACCEPT for 2204. SSH config for that host exists in the agent container; **no sshfs mount yet**. See [`../stella-server/wordpress-staging.md`](../stella-server/wordpress-staging.md).

See [`../stella-server/infrastructure.md`](../stella-server/infrastructure.md) (DOCKER-USER script) and [`../stella-server/dev-ssh-access.md`](../stella-server/dev-ssh-access.md) (`edison` user).

---

## Adding another remote project

Same pattern as the first mount — new folder, new SSH config host, Stella-side user/port as needed:

1. Dedicated agent key already exists (`edison_agent`); add its pubkey to the new target.
2. Confirm Stella (or other host) firewall ACCEPTs `edison-ip` (`178.105.203.54`) on that SSH port.
3. Mount at `/mnt/<full-hostname>` inside the agent container — see [cursor-agent.md](cursor-agent.md).
