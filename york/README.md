# york — WireGuard access gateway

**Status:** Set up 2026-11-07. Access gateway, not a service host. Operator traffic to Stella / Atlas / Edison leaves Hetzner via the **floating IP** `78.47.218.190` (the whitelisted exit). Proton `194.126.177.181` stays as fallback everywhere it was already allowed.

**Related:** Stella firewall [`../stella-server/infrastructure.md`](../stella-server/infrastructure.md), Atlas firewall [`../atlas/server.md`](../atlas/server.md), Edison firewall [`../edison/infrastructure.md`](../edison/infrastructure.md), company map [`../integration/system-overview.md`](../integration/system-overview.md). Atlas monitoring: `/monitoring/york` (Hetzner Cloud API only — no health-api). Open items: [`../open-gaps.md`](../open-gaps.md) (section york).

---

## Why (2026-11-07)

Cursor Remote-SSH to `atlas.dev` kept dropping: “Workspace Disconnected”, ssh “Timeout, server 95.217.144.93 not responding”, `client_loop: send disconnect: Broken pipe`.

`ping -c 20 95.217.144.93`:

| Path | Loss |
|------|------|
| Proton WireGuard (UDP) | 55% |
| Proton WireGuard (TCP) | 35% |
| No VPN | 0% |

Switching the Proton protocol did not help. The loss is between Proton’s exit (dedicated IP `194.126.177.181`) and Hetzner, not on the home line.

Stella was healthy at the time: cursor-server running 10+ days, `-ssh` container 1.2 GB RAM / ~9% CPU, inotify limit ~1M. The Cursor agent’s claim that the disconnects came from the Vite black-screen problem was wrong; those are two separate issues. Black-screen notes: [`../atlas/dev-instance.md`](../atlas/dev-instance.md).

**Fix:** own WireGuard server in Hetzner Cloud. Result through york: **0% loss**, ~35 ms (Proton: ~118 ms).

---

## Host

| | |
|---|---|
| **Name** | `york` |
| **Provider** | Hetzner Cloud CX23, Falkenstein |
| **OS** | Ubuntu 26.04 (`resolute`) |
| **Hetzner Cloud project** | **TODO (Dominik)** |
| **Primary IP** | Hetzner name **`york-ip`**, address **`2.28.33.52`**. **Not whitelisted** anywhere. Test / general exit IP. Own traffic (apt etc.) leaves via this address. |
| **Floating IP** | Hetzner name **`york-floating-ip`**, address **`78.47.218.190`**. **The whitelisted exit IP.** Chosen because it can move to a replacement server without touching any whitelist. |
| **WireGuard** | `wg0`, `10.8.0.1/24`, UDP/51820 |

Floating IP is configured permanently in `/etc/netplan/60-floating-ip.yaml` (`eth0`, `78.47.218.190/32`, chmod 600, `netplan apply`).

### Hetzner firewall `york-firewall`

| Rule | Source |
|------|--------|
| TCP/22 | `194.126.177.181/32` only (Proton, fallback) |
| ICMP | any |
| UDP/51820 | any IPv4 (no IPv6 rule) |

The home IP is dynamic and is **not** whitelisted.

Access:

| Path | How |
|------|-----|
| Normal | `ssh york` through the tunnel (`HostName 10.8.0.1`) |
| Fallback | `ssh york-public` via Proton (`HostName 2.28.33.52`) |
| Last resort | Hetzner web console |

Root SSH key: Dominik’s MacBook key (`ssh-ed25519 … dominik@Dominiks-MacBook-Pro.local`), added via `ssh-copy-id` after a first login with the Hetzner root password. Do not commit the key.

Password login disabled: `/etc/ssh/sshd_config.d/01-no-password.conf` (`PasswordAuthentication no`).

**Gotcha:** the file must sort **before** `50-cloud-init.conf` (which sets `yes`; sshd uses the first value). First attempt as `99-…` did not work. Verify with `sshd -T | grep -i passwordauthentication`.

Setup: `apt update && apt full-upgrade -y`, `unattended-upgrades` enabled (verified `/etc/apt/apt.conf.d/20auto-upgrades` has both values `"1"`). No fail2ban (SSH is IP-restricted + key-only; WireGuard does not answer without a valid key).

`net.ipv4.ip_forward=1` in `/etc/sysctl.d/99-wireguard.conf`. IPv6 is **not** forwarded.

---

## WireGuard

`/etc/wireguard/wg0.conf`, interface `10.8.0.1/24`, port 51820, `systemctl enable wg-quick@wg0`.

### Peers

| Peer | Tunnel IP | Leaves via | Restrictions |
|---|---|---|---|
| dominik | 10.8.0.2 | 78.47.218.190 | none |
| test | 10.8.0.3 | 2.28.33.52 | none (non-whitelisted exit, also used for other projects) |
| pawel | 10.8.0.4 | 78.47.218.190 | FORWARD to atlas `2.31.42.13` dropped; INPUT on wg0 (york itself) dropped |

iptables chains `WG_NAT` (nat table, SNAT per source IP) and `WG_FWD` (filter). Rules live as one `PostUp =` / `PostDown =` line per iptables command directly in `wg0.conf`. `/etc/wireguard/up.sh` and `down.sh` exist only as readable source.

**Gotcha Ubuntu 26.04:** `PostUp = /etc/wireguard/up.sh` (exit 126) and `PostUp = /bin/sh …` (`/bin/sh: Permission denied`) both fail when wg-quick runs them. Most likely AppArmor confinement of wg-quick (**UNVERIFIED**). Calling `iptables` directly works. Debug with `wg-quick up wg0` (prints every step) instead of `systemctl`.

WireGuard was purged and reinstalled once on 2026-11-07 (`apt purge wireguard wireguard-tools`, `rm -rf /etc/wireguard`) while debugging that. Keys were also rotated that day after the first set was exposed in a chat. Current keys are from the reinstall. Do not commit keys.

### Client configs

`/etc/wireguard/clients/{dominik,test,pawel}.conf` (root-only). All full tunnel:

| Field | Value |
|-------|-------|
| `AllowedIPs` | `0.0.0.0/0, ::/0` |
| `DNS` | `9.9.9.9` |
| `Endpoint` | `2.28.33.52:51820` |
| `PersistentKeepalive` | `25` |

Stored in Dominik’s password manager. Fetch with `scp york:/etc/wireguard/clients/<name>.conf .` — do not commit the files.

`::/0` blackholes IPv6 on the client (york does not forward IPv6) to avoid leaks. See [`../open-gaps.md`](../open-gaps.md).

---

## Client side (MacBook)

- Client app: Defguard (import the `.conf`; tunnel name “York”).
- `~/.ssh/config`: `Host york` → `HostName 10.8.0.1`, `User root`; `Host york-public` → `HostName 2.28.33.52`, `User root`.
- `~/.ssh/config` also got `Host *` → `ServerAliveInterval 15`, `ServerAliveCountMax 4` (helps with idle drops through VPN/NAT).
- Only one VPN at a time: Proton and the york tunnel conflict on macOS.
- Full tunnel = websites see a Hetzner datacenter IP (`78.47.218.190`). Not anonymous like Proton; more captchas possible. If york is down while the tunnel is active, there is no internet until the tunnel is disconnected.

---

## Whitelist rule (2026-11-07)

Add `78.47.218.190` next to `194.126.177.181`. Proton stays as fallback everywhere.

What was updated that day (Stella `DOCKER-USER` + UFW/22 + three Caddy allow lists; Atlas Hetzner firewall + `ATLAS_ALLOWED_IPS`; Edison Hetzner firewall) is recorded on those pages. What was **not** updated is in [`../open-gaps.md`](../open-gaps.md) (Atlas Connect `autologin_ip_whitelist`, Edison host-api `/v1/*`, `api.atlas` Caddy block, ddashboard `api/v1`).

Verified through york: `ssh atlas.dev.foxcraft.digital 'echo OK'` → OK, and `https://atlas.dev.foxcraft.digital` → 200.
