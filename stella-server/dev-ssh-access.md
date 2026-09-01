# Dev SSH Access — Per-App SSH Containers for Cursor Remote-SSH

**Status:** Implemented 2026-09-01 for `osgar-datahub` (port 2201) and `advoapp.finditoo` (port 2202). Closes the open-gaps TODO "User-based SSH access on Stella."

**Purpose:** give individual devs (Dominik, Pawel) Cursor Remote-SSH access scoped to a single app's source, without exposing the host or other apps/services. Root SSH remains Dominik-only, unchanged.

**Related**

- Host topology, `DOCKER-USER`, core compose: [`infrastructure.md`](infrastructure.md)
- .NET app folder / `-dev` pattern: [`dotnet-app-deployment.md`](dotnet-app-deployment.md)

---

## Why not host-level Linux users

The original TODO envisioned per-user Linux accounts with `authorized_keys` on the host. Rejected in favor of per-app containers because:

- A host user, even with restricted permissions, still sees the whole filesystem tree and can enumerate other apps/services.
- Devs need real interactive SSH (Cursor Remote-SSH installs a server binary, needs SFTP + port forwarding) — a `docker exec` forced-command approach (considered and rejected) breaks this, since forced commands block port forwarding.
- A container with only the relevant app's source bind-mounted gives hard filesystem isolation for free.

## Architecture

One extra service per app, alongside the existing `-dev` container, in the same `docker-compose.dev.yml`:

```
/opt/apps/dotnet/<subdomain>/
  ├── Dockerfile.dev          # existing, hot-reload dev container
  ├── Dockerfile.ssh          # NEW — sshd + dev tooling
  ├── authorized_keys         # NEW — both devs' public keys, one file
  ├── docker-compose.dev.yml  # existing file, new service appended
  └── src/                    # existing, now group-writable (see Permissions below)
```

### Dockerfile.ssh (template)

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:10.0

RUN apt-get update && apt-get install -y --no-install-recommends \
    openssh-server curl wget ca-certificates git \
    && mkdir /var/run/sshd \
    && rm -rf /var/lib/apt/lists/*

RUN useradd -m -s /bin/bash -G $(getent group 1000 | cut -d: -f1) dominik \
    && useradd -m -s /bin/bash -G $(getent group 1000 | cut -d: -f1) pawel

RUN mkdir -p /home/dominik/.ssh /home/pawel/.ssh
COPY authorized_keys /home/dominik/.ssh/authorized_keys
COPY authorized_keys /home/pawel/.ssh/authorized_keys

RUN chmod 700 /home/dominik/.ssh /home/pawel/.ssh \
    && chmod 600 /home/dominik/.ssh/authorized_keys /home/pawel/.ssh/authorized_keys \
    && chown -R dominik:dominik /home/dominik/.ssh \
    && chown -R pawel:pawel /home/pawel/.ssh

EXPOSE 22
CMD ["/usr/sbin/sshd", "-D"]
```

**`getent group 1000` instead of `groupadd -g 1000 devs`:** the `mcr.microsoft.com/dotnet/sdk` base image already has a GID 1000 group (from its own default non-root user setup). Creating a new group at the same GID fails (`groupadd: GID '1000' already exists`) — reuse it instead. One `authorized_keys` file is copied to both users; either dev's key can log in as either Linux user inside the container (acceptable — the isolation boundary is per-app, not per-dev-identity within an app).

**Both users get the same key file deliberately** — simpler than maintaining two files, and the security boundary that matters is app-to-app, not dev-to-dev within one app's container.

### Compose service (template)

```yaml
  <app>-ssh:
    build:
      context: .
      dockerfile: Dockerfile.ssh
    container_name: <app>-ssh
    volumes:
      - ./src:/home/dominik/app
      - ./src:/home/pawel/app
    ports:
      - "22XX:22"
    restart: unless-stopped
```

**No `networks:` key.** This is the critical line. Do not add `- edge`. Omitting `networks:` puts the container in the Compose project's own auto-created `<project>_default` network only.

**Both volume lines point at the same host folder** (`./src`), just mounted at two different container paths — Dominik and Pawel share one working copy per app, not separate checkouts. This was a deliberate choice, confirmed with Dominik; separate checkouts would need `./src-pawel:/home/pawel/app` with its own folder instead.

## Network isolation — what "isolated" actually means here

Confirmed via a live Cursor-agent probe from inside each container (2026-09-01):

| Container | Can reach | Cannot reach |
|---|---|---|
| `osgar-datahub-ssh` | `osgar-datahub-db:1433` (own SQL Server), `osgar-datahub-dev:8080` (own app) | Finditoo, Caddy, `deploy-api`, `health-api`, Ollama, any `edge`-network service |
| `advoapp-ssh` | nothing else in this compose file (no sibling DB defined) | same list as above; outbound public HTTPS still works (e.g. `app.finditoo.foxcraft.digital` resolves and responds — that's normal internet egress, not a network leak) |

**Why an app's own DB is reachable:** Compose auto-creates one network per project (per `docker-compose.dev.yml` file) and joins every service that doesn't declare `networks:` explicitly, *and* every service that does declare `- default`. Since `osgar-datahub-db` and `osgar-datahub-dev` both list `default` explicitly, and `osgar-datahub-ssh` has no `networks:` key at all, all three land in the same project-level `default` network and can see each other by container name. Apps defined in *other* compose files (different project folders) get their own separate `default` network — hence no cross-app visibility.

**This is intentional, confirmed with Dominik 2026-09-01:** an app's SSH/dev container is allowed to see that same app's own database — useful for debugging — but must never see another app or any core Stella service (`edge` network). Do not add `edge` to any future `-ssh` service; that was the actual cause of the first (fixed) leak below.

### Incident: first `osgar-datahub-ssh` build leaked onto `edge`

The very first version of `osgar-datahub-ssh` was built with `networks: [edge]` (copy-pasted from the `-dev` pattern, which legitimately needs `edge` so Caddy can reach it by name). A Cursor-agent probe run from inside that container found and connected to: Finditoo (`172.20.0.4:8080`, hit a login redirect but got a response), Caddy (`172.20.0.3:80/443`), `deploy-api` (`172.20.0.2:8080`, `/health`, `/docs`, `/apps` all reachable unauthenticated — mutating routes correctly rejected unsigned requests), and `health-api` (`172.20.0.7:8080`, same). Fixed by removing `networks: [edge]` entirely and recreating the container — re-probing afterward confirmed the container dropped into an isolated per-project `_default` network with none of the above reachable. **Any new `-ssh` service must omit `networks:` entirely — this is not optional.**

## Filesystem permissions — the UID/GID problem

App source folders on the host are owned `root:root`, mode `755` (normal for these apps — the `-dev` containers run as root, so this was never an issue before). The new `dominik`/`pawel` users inside the `-ssh` container are non-root and, without a fix, get `Permission denied` on every write — confirmed by a live Cursor-agent write test before the fix.

**Fix, applied per app before building the `-ssh` container:**

```bash
sudo chgrp -R 1000 /opt/apps/dotnet/<subdomain>/src
sudo chmod -R g+rwX /opt/apps/dotnet/<subdomain>/src
sudo find /opt/apps/dotnet/<subdomain>/src -type d -exec chmod g+s {} \;
```

- `chgrp -R 1000` — group-owns the whole tree to GID 1000 (the same GID both `dominik` and `pawel` are secondary-group members of inside the container, via the `getent group 1000` trick above). Does not change the `root` owner, only adds group access.
- `chmod -R g+rwX` — group gets read/write on files, read/write/execute on directories (capital `X` = execute only if already set for someone, i.e. directories get traversable, files don't become spuriously executable).
- `find ... chmod g+s` (SetGID bit on every directory) — **required**, not optional. Without it, new files/folders created inside the container (e.g. `dotnet` build output, new source files) inherit the creating user's primary group (`dominik`/`pawel`), not group 1000, and lose write access for the other dev. SetGID forces new entries to inherit the parent directory's group.

Confirmed working via live write test inside Cursor: `touch ~/app/test-write.txt && rm ~/app/test-write.txt` succeeded after the fix, failed before it.

## Firewall — same `DOCKER-USER` pattern as `imap-sync`

Each `-ssh` service publishes its port on `0.0.0.0` (`"22XX:22"`), which — per [`infrastructure.md`](infrastructure.md) — bypasses UFW entirely. Must be locked down the same way as `imap-sync`'s (now-retired) port 3001: append to `/usr/local/bin/docker-user-firewall.sh` (systemd-applied on boot, reapplied manually with `sudo /usr/local/bin/docker-user-firewall.sh`):

```bash
iptables -A DOCKER-USER -i $WAN_IF -s 194.126.177.181 -p tcp --dport 22XX -j ACCEPT
iptables -A DOCKER-USER -i $WAN_IF -s 23.88.90.12 -p tcp --dport 22XX -j ACCEPT
iptables -A DOCKER-USER -i $WAN_IF -p tcp --dport 22XX -j DROP
```

**Ports allocated so far:**

| Port | App |
|---|---|
| 2201 | osgar-datahub |
| 2202 | advoapp (finditoo) |

Next app gets 2203, and so on. Update the table above when adding one.

**Verified active** via `iptables -L DOCKER-USER -n -v --line-numbers` — both ACCEPT rules for the two whitelisted IPs, DROP for everyone else, correct order (ACCEPT before DROP, per the three-iteration lesson already documented in [`infrastructure.md`](infrastructure.md)).

## Client setup (Cursor)

`~/.ssh/config` (Mac: `~/.ssh/config`; Windows: `C:\Users\<user>\.ssh\config`):

```
Host osgar-datahub
    HostName 95.217.144.93
    Port 2201
    User dominik
    IdentityFile ~/.ssh/id_ed25519

Host advoapp-finditoo
    HostName 95.217.144.93
    Port 2202
    User dominik
    IdentityFile ~/.ssh/id_ed25519
```

Pawel: same file, `User pawel`, his own `IdentityFile`. Windows devs need `ssh-keygen -t ed25519` first if no key exists yet (PowerShell), then their public key added to the relevant apps' `authorized_keys` files (rebuild the `-ssh` container after).

Cursor: `Cmd/Ctrl+Shift+P` → "Remote-SSH: Connect to Host" → pick the host alias. Cursor auto-installs its remote server on first connect (~30-60s). Confirmed working end-to-end for both apps, including live file edits without copying, and `git` operations (needs `git config --global --add safe.directory /home/dominik/app` inside the container first, since the repo is owned `root:ubuntu` and the SSH user is not root — noted by the Cursor agent during testing, not yet pre-baked into the Dockerfile).

## Verifying isolation — external checklist

A standalone script exists for manually confirming no port is reachable from a non-whitelisted IP (run from a phone on mobile data, never from a whitelisted connection): `stella-firewall-check.sh` (not yet committed to this repo — ask Dominik for the current copy, or regenerate: it's a simple loop of `curl --connect-timeout 5` against each public port, expecting timeouts). Extend its list with each new `22XX` port as apps are added.

## Adding SSH to a new app — checklist

1. Add `Dockerfile.ssh` + `authorized_keys` next to `Dockerfile.dev` (template above).
2. Fix host `src/` group perms (`chgrp 1000` / `g+rwX` / SetGID) **before** first connect.
3. Append the `-ssh` service to `docker-compose.dev.yml` — **no `networks:` key**, unique host port `22XX`.
4. Append the three `DOCKER-USER` rules for that port; re-run `sudo /usr/local/bin/docker-user-firewall.sh`.
5. `docker compose -f docker-compose.dev.yml up -d --build <app>-ssh`
6. Update the port table in this doc and the client's `~/.ssh/config`.
7. Probe from a non-whitelisted IP (or `stella-firewall-check.sh`) that the new port times out for everyone else.

---
