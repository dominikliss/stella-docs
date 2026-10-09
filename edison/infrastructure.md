# Edison — Infrastructure

**Status:** Rebuilt 2026-10-09 (old CPX32 renamed `edison-old`, then deleted). HTTP/HTTPS not opened yet.

**Related:** [README.md](README.md) (role + naming), [cursor-agent.md](cursor-agent.md) (agent container)

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
| TCP/22 (SSH) | Trusted IP `194.126.177.181` | Same laptop/VPN IP used on Stella |
| ICMP | Same trusted IP — **not** Any | Ping restricted the same way as SSH |
| TCP/80, TCP/443 | none | Open only when a web app is exposed; keep IP-restricted |

**Verified 2026-10-09** from Stella (`95.217.144.93`): port 22 on `178.105.203.54` is blocked.

This firewall does **not** see Docker-internal traffic. Containers on `edison-net` (web app → `cursor-agent`, etc.) talk to each other by container name and never hit these rules.

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

Docker **29.9.0**, installed with the official script (not distro packages):

```bash
curl -fsSL https://get.docker.com | sh
```

User-defined bridge so every agent (and the future web app) share Docker DNS:

```bash
docker network create edison-net
```

All agent and control-plane containers join `--network edison-net`. Edison appears in the **network** name only. Container and host-folder names follow the **tool**: `cursor-agent`, `~/cursor-agent/`, later `opencode-agent` / `~/opencode-agent/`.

Only port 22 is listening publicly; the agent container publishes no ports.

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
