# Edison — Infrastructure

**Status:** Provisioned 2026-09-22. HTTP/HTTPS not opened yet.

**Related:** [README.md](README.md) (role + naming), [cursor-agent.md](cursor-agent.md) (agent container)

---

## Server

| | |
|---|---|
| **Name** | Edison |
| **Provider** | Hetzner Cloud |
| **Type** | CPX32 — 4 vCPU shared, 8 GB RAM, 160 GB SSD |
| **OS** | Ubuntu 26.04 LTS |
| **Location** | Falkenstein |
| **Primary IP** | Dedicated address, Hetzner name `dev-agent-ip` (functional name). Stella `DOCKER-USER` ACCEPTs `178.105.203.54` on ports 2201 and 2204 — **ASSUMED** to be this IP. **TODO (Dominik):** confirm, then record the numeric IP here. |
| **Backups** | Hetzner automatic backups **off** on purpose (sandbox; code lives in git). Take a manual snapshot before risky host changes. |

Shared (CPX) rather than dedicated (CCX) is intentional: one or two interactive agent sessions do not justify CCX pricing. Resize to CPX42 / CPX52 is a console click if load grows.

Edison must stay off Stella. Do not colocate agent containers on the Stella dedicated box.

---

## Firewall (Hetzner Cloud)

Inbound, public NIC only:

| Rule | Source | Notes |
|------|--------|--------|
| TCP/22 (SSH) | Trusted IP `194.126.177.181` | Same laptop/VPN IP used on Stella |
| ICMP | Same trusted IP — **not** Any | Ping restricted the same way as SSH |
| TCP/80, TCP/443 | none | Open only when a web app is exposed; keep IP-restricted |

This firewall does **not** see Docker-internal traffic. Containers on `edison-net` (web app → `cursor-agent`, etc.) talk to each other by container name and never hit these rules.

---

## SSH onto Edison (the host)

- User: `root`
- Auth: SSH key selected at server create. Reused the key originally created for Stella, or a dedicated one — **record the actual key comment/filename here once confirmed.**
- Local shortcut on the MacBook (`~/.ssh/config`):

```
Host edison
    HostName <dev-agent-ip>
    User root
    IdentityFile ~/.ssh/<key-used-at-create>
```

Then: `ssh edison`.

---

## Docker on the host

Installed with the official script (not distro packages):

```bash
curl -fsSL https://get.docker.com | sh
```

User-defined bridge so every agent (and the future web app) share Docker DNS:

```bash
docker network create edison-net
```

All agent and control-plane containers join `--network edison-net`. Edison appears in the **network** name only. Container and host-folder names follow the **tool**: `cursor-agent`, `~/cursor-agent/`, later `opencode-agent` / `~/opencode-agent/`.

---

## Stella-side access (required for sshfs)

The agent SSHes to Stella’s per-app SSH container, not to Stella’s host SSH:

| | |
|---|---|
| Host | `osgar.datahub.foxcraft.digital` (Stella public IP `95.217.144.93` at setup) |
| Port | **2201** (published by `osgar-datahub-ssh`) |
| User | `edison` — dedicated user inside that container, for revoke-without-touching-devs |

Stella’s `DOCKER-USER` chain ACCEPTs `194.126.177.181`, `23.88.90.12`, and `178.105.203.54` on 2201 (and the same three on 2204) and DROPs everything else. `178.105.203.54` is **ASSUMED** to be Edison’s public IP (`dev-agent-ip`); **TODO (Dominik):** confirm and record it in the table at the top of this page.

A second per-app SSH container with an `edison` user now exists on port **2204** (`dominikliss.foxcraft.digital`). It is reachable only if the Edison IP is in the `DOCKER-USER` ACCEPT for 2204 (currently `178.105.203.54`). See [`../stella-server/wordpress-staging.md`](../stella-server/wordpress-staging.md).

See [`../stella-server/infrastructure.md`](../stella-server/infrastructure.md) (DOCKER-USER script) and [`../stella-server/dev-ssh-access.md`](../stella-server/dev-ssh-access.md) (`edison` user).

---

## Adding another remote project

Same pattern as the first mount — new folder, new SSH config host, Stella-side user/port as needed:

1. Dedicated agent key already exists (`edison_agent`); add its pubkey to the new target.
2. Confirm Stella (or other host) firewall ACCEPTs `dev-agent-ip` on that SSH port.
3. Mount at `/mnt/<full-hostname>` inside the agent container — see [cursor-agent.md](cursor-agent.md).
