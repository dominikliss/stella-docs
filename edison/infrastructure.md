# Edison — Infrastructure

**Status:** Rebuilt 2026-10-09 (old CPX32 renamed `edison-old`, then deleted). TCP/443 opened 2026-11-01 for `api.edison.foxcraft.digital` (IP-restricted; Stella allowed, verified 2026-11-02). Port 80 stays closed. Workspaces v1 mounts: osgar, dominikliss.

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
| **Primary IP** | Hetzner name **`edison-ip`**, address **`178.105.203.54`** (Falkenstein). Moved from the old server to this one. From Edison this IP reaches Stella SSH ports 2201, 2202, 2204, 2205, 2206, 2207 (verified 2026-11-02). **TODO (Dominik):** confirm the exact `DOCKER-USER` lines for 2202 and 2206. |
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
| TCP/443 | `194.126.177.181`, `95.217.144.93` (Stella: Atlas dev, edison.dev), `2.31.42.13` (Atlas prod) | Opened 2026-11-01 for `api.edison.foxcraft.digital`. Caddy publishes 443 (`docker-compose.yml`: `caddy` with `443:443`). `/v1/*` is IP-allowlisted; Stella is allowed (verified by `GET /v1/capabilities` → 200 from Stella, 2026-11-02). Port 80 not opened. **TODO (Dominik):** record the exact Caddy allowlist and the Hetzner Cloud firewall rule for 443 as they stand today. |

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

Named volume `host-api-state` → `/var/lib/host-api` (`sessions.json`, `mounts.json`, `locks/`).

`ss` on the host shows **22** and **443**. Caddy publishes 443 for `api.edison.foxcraft.digital`. The agent container publishes no ports. `ufw` is inactive — the Hetzner Cloud firewall is the only filter. See [host-api.md](host-api.md). **UNVERIFIED:** the exact live Hetzner Cloud firewall rule set for 443. **TODO (Dominik):** record it.

**Pre-checks 2026-11-01** (before this setup): no API, no Caddy, nothing listening except sshd (22) and systemd-resolved (`127.0.0.53`/`54`). 75 GB disk / 7.6 GB RAM, no swap. `~/cursor-agent` unchanged.

---

## Stella-side access (required for sshfs)

The agent SSHes to Stella’s per-app SSH container, not to Stella’s host SSH. Test 2026-11-02: `ssh -o BatchMode=yes -p <port> edison@<host>` from inside `cursor-agent` with the `edison_agent` key.

| Host | Port | Port reachable from Edison | `edison` key login | `~/app` |
|---|---|---|---|---|
| osgar.datahub | 2201 | yes | yes | `/home/edison/app` (.NET tree) |
| advoapp.finditoo | 2202 | **yes** (docs did not record an Edison ACCEPT) | **no** — `Permission denied (publickey,password)` | — |
| dominikliss | 2204 | yes | yes | WordPress root |
| atlas.dev | 2205 | yes | yes | Laravel root |
| hypershape | 2206 | yes (source IPs were undocumented) | yes | WordPress root |
| edison.dev | 2207 | yes | yes | Laravel root |

The `edison_agent` public key is present on 2205, 2206 and 2207 too, not only on 2201/2204. The first connection to advoapp added its host key to `~/cursor-agent/known_hosts`.

**TODO (Dominik):** confirm the exact `DOCKER-USER` lines for 2202 and 2206 in `/usr/local/bin/docker-user-firewall.sh`.

See [`../stella-server/infrastructure.md`](../stella-server/infrastructure.md) (DOCKER-USER script) and [`../stella-server/dev-ssh-access.md`](../stella-server/dev-ssh-access.md) (`edison` user).

---

## Adding another remote project

1. Add a row to `/opt/services/host-api/catalog.json` (host-owned; re-read on every request).
2. Stella: `edison` user + `edison_agent` pubkey in that app’s SSH container.
3. Stella: `DOCKER-USER` ACCEPT for `edison-ip` (`178.105.203.54`) on that SSH port.
4. `POST /v1/workspaces/ensure` with the catalog FQDN.

No manual mount and no ssh-config entry needed. The API passes every sshfs option on the command line. See [host-api.md](host-api.md) and [cursor-agent.md](cursor-agent.md).
