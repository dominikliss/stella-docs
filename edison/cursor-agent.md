# Edison — Cursor Agent Container

**Status:** First container running 2026-09-22. Cursor CLI `2026.09.18-9a7762b`. `sshfs` + `cursor-agent` verified against `osgar.datahub.foxcraft.digital`.

**Related:** [README.md](README.md), [infrastructure.md](infrastructure.md), Stella [`dev-ssh-access.md`](../stella-server/dev-ssh-access.md)

---

## Naming

| Thing | Pattern | Example |
|-------|---------|---------|
| Container | tool name, not server name | `cursor-agent` (later `opencode-agent`, `claude-code-agent`) |
| Host folder | same as container | `~/cursor-agent/` |
| Image | tool + `-image` | `cursor-agent-image` |
| Docker network | host name | `edison-net` |
| Remote mount | `/mnt/<full-hostname>` | `/mnt/osgar.datahub.foxcraft.digital` |

---

## Dockerfile

`~/cursor-agent/Dockerfile` — include `openssh-client` and `sshfs` in the image. Both were discovered as required after the first build; do not install them ad-hoc in a running container.

```dockerfile
FROM ubuntu:24.04

RUN apt update && apt install -y curl git ca-certificates openssh-client sshfs

RUN curl https://cursor.com/install -fsS | bash

ENV PATH="/root/.local/bin:${PATH}"

WORKDIR /workspace
```

Build:

```bash
cd ~/cursor-agent
docker build -t cursor-agent-image .
```

---

## Run flags

```bash
docker run -it --name cursor-agent --network edison-net \
  --cap-add SYS_ADMIN --device /dev/fuse \
  --security-opt apparmor:unconfined \
  --security-opt seccomp:unconfined \
  -v ~/.ssh/edison_agent:/root/.ssh/id_ed25519:ro \
  -v ~/.ssh/edison_agent.pub:/root/.ssh/id_ed25519.pub:ro \
  cursor-agent-image bash
```

### Why these flags exist

| Flag | What fails without it |
|------|------------------------|
| `--cap-add SYS_ADMIN --device /dev/fuse` | `sshfs`: `fuse: device not found, try 'modprobe fuse' first` |
| `--security-opt apparmor:unconfined --security-opt seccomp:unconfined` | `sshfs` still `Permission denied` — Docker’s default seccomp profile blocks the `mount` syscall |
| `-v ~/.ssh/edison_agent:…` | Agent has no key for remote SSH |

**Volume-mount gotcha:** if the host path does not exist, Docker creates an empty **directory** at that path and mounts it. Inside the container `ls -la` then shows `drwxr-xr-x` instead of a key file. Always `ls` the host file before `docker run`.

---

## Agent SSH key (Edison → remotes)

Dedicated key, generated **on the Edison host** (not inside the container, not a reused laptop/Stella key). Separate key so Edison/agent access can be revoked without touching other systems.

```bash
ssh-keygen -t ed25519 -C "edison-agent" -f ~/.ssh/edison_agent
```

- Host filenames: `~/.ssh/edison_agent` / `edison_agent.pub`
- Inside the container the same files are mounted at the default SSH paths `/root/.ssh/id_ed25519` and `.pub` — names differ on purpose
- Pubkey installed on targets. First target: `osgar.datahub.foxcraft.digital`, user `edison` (see [`../stella-server/dev-ssh-access.md`](../stella-server/dev-ssh-access.md))

### SSH config inside the container

`/root/.ssh/config` (today **only in the running container** — lost on recreate; persist via a host mount, open gap):

```
Host osgar.datahub.foxcraft.digital
    User edison
    Port 2201
    IdentityFile /root/.ssh/id_ed25519
```

Then `ssh osgar.datahub.foxcraft.digital` needs no extra flags.

---

## Working model: sshfs, not a local checkout

The agent does **not** clone the repo onto Edison. It works live on the remote tree, same idea as Cursor IDE Remote-SSH.

```bash
mkdir -p /mnt/osgar.datahub.foxcraft.digital
sshfs edison@osgar.datahub.foxcraft.digital:/home/edison/app \
  /mnt/osgar.datahub.foxcraft.digital -p 2201
```

(`-p 2201` is redundant if the SSH config above is present.)

This is a live mount, not a copy. There is no second tree on Edison. Tools (`ls`, `grep -r`, `cursor-agent`) see a normal directory.

**Verified 2026-09-22:** `ls`, `grep -r`, and `cursor-agent` started in the mount successfully read project layout, README, `.csproj` files.

Further remotes: another `/mnt/<full-hostname>`. Same key, same network, new SSH config host.

---

## Cursor CLI — auth and first-run

- Version at setup: `2026.09.18-9a7762b` (re-check `cursor-agent --help` after upgrades; login/headless flags move)
- Login: `cursor-agent login` (Cursor account). Confirm the flow on the next doc pass if the version changed.
- First start in a new directory: **Workspace Trust Required** — answer `a` (Trust this workspace).
- CLI is a local process. It does not install a remote Cursor server. Use `sshfs` for remote files.

### Allowlist (shell commands the agent wants to run)

Prompt per unseen command: run once (`y`), add to allowlist (`Tab`), or allow all (`Shift+Tab`).

**On the allowlist** (read-only / structurally harmless):

- `head`
- `git rev-parse`
- `date`
- `grep`
- `git ls-files`

(`grep` / `git ls-files` were added **without** bundling `kill` in the same confirmation.)

**Confirmed per call, not allowlisted:**

| Command | Why not blanket-allow |
|---------|------------------------|
| `kill` | Too powerful for a standing grant — even when the live case was only the agent’s own hung child. Matters more once a web app triggers the agent unattended. |
| `python` / `python3` | Generic interpreter; the allowlist matches the binary, not the script body. |

**Rule for later additions:** allow commands that can only read, regardless of arguments. Confirm individually anything that is a generic interpreter or that changes/kills state (`rm`, `kill`, `python`, …). Same rule applies harder once runs are headless.

---

## Staging access — decision

The container has the same reach into staging as the previous laptop Cursor IDE Remote-SSH setup. That is intentional.

When the planned web app starts runs (instead of a person at a TTY), add logging/audit of what the agent does on staging.

---

## Adding another agent (checklist)

1. `~/<tool>-agent/Dockerfile` + image `<tool>-agent-image`.
2. `docker run --name <tool>-agent --network edison-net` with the same FUSE/seccomp flags if that tool also uses `sshfs`.
3. Reuse `/mnt/<full-hostname>` mounts (or remount in the new container). Reuse `edison_agent` unless a tool needs its own key.
4. Do not name the container `edison-*`.

---

## Persistence still missing

| Path | Today | Should be |
|------|--------|-----------|
| Agent private key | Host file, bind-mounted | keep |
| `/root/.ssh/config` | Only in container writable layer | host bind-mount, same as the key |
| sshfs mounts | Manual after each start | document a start snippet or entrypoint that remounts `/mnt/*` |
