# Edison — Cursor Agent Container

**Status:** Rebuilt 2026-10-09 on the new CX33 host. `sshfs` + headless `cursor-agent` verified against `osgar.datahub.foxcraft.digital`. Persistence (ssh config, known_hosts, Cursor login and CLI state) is on the host. `dominikliss` has an SSH config entry but no mount yet.

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

Everything below lives on the Edison host under `~/cursor-agent/` unless noted.

**Do not commit `~/cursor-agent` or `state/`.** `~/cursor-agent/state/config-cursor/auth.json` is the Cursor login token — treat as secret; never write its contents into docs.

---

## Dockerfile

`~/cursor-agent/Dockerfile` — include `openssh-client` and `sshfs` in the image. Plus wrapper `/usr/local/bin/cursor-agent-run` (`umask 002` then `exec cursor-agent "$@"`). Use the wrapper instead of `cursor-agent` for any run that may write files (also for the future web app).

```dockerfile
FROM ubuntu:24.04

RUN apt update && apt install -y curl git ca-certificates openssh-client sshfs

RUN curl https://cursor.com/install -fsS | bash

ENV PATH="/root/.local/bin:${PATH}"

WORKDIR /workspace
```

Wrapper `/usr/local/bin/cursor-agent-run` (shell script in the image):

```sh
umask 002
exec cursor-agent "$@"
```

Build:

```bash
cd ~/cursor-agent
docker build -t cursor-agent-image .
```

Anything added by hand inside the container is lost at reboot; changes belong in the Dockerfile or `run.sh`.

---

## run.sh

Recreates the container and mounts sshfs. Invoked by systemd at every boot.

```bash
docker rm -f cursor-agent
docker run -dit --name cursor-agent --network edison-net \
  --restart unless-stopped \
  --cap-add SYS_ADMIN --device /dev/fuse \
  --security-opt apparmor:unconfined \
  --security-opt seccomp:unconfined \
  -v ~/.ssh/edison_agent:/root/.ssh/id_ed25519:ro \
  -v ~/.ssh/edison_agent.pub:/root/.ssh/id_ed25519.pub:ro \
  -v ~/cursor-agent/ssh-config:/root/.ssh/config:ro \
  -v ~/cursor-agent/known_hosts:/root/.ssh/known_hosts \
  -v ~/cursor-agent/state/cursor:/root/.cursor \
  -v ~/cursor-agent/state/config-cursor:/root/.config/cursor \
  cursor-agent-image
```

Then inside the container:

```bash
mkdir -p /mnt/osgar.datahub.foxcraft.digital
sshfs -o reconnect,ServerAliveInterval=15,ServerAliveCountMax=3 \
  edison@osgar.datahub.foxcraft.digital:/home/edison/app \
  /mnt/osgar.datahub.foxcraft.digital
```

Only osgar is mounted. `dominikliss.foxcraft.digital` has an SSH config entry but **no mount yet**.

### Why these flags exist

| Flag | What fails without it |
|------|------------------------|
| `--cap-add SYS_ADMIN --device /dev/fuse` | `sshfs`: `fuse: device not found, try 'modprobe fuse' first` |
| `--security-opt apparmor:unconfined --security-opt seccomp:unconfined` | `sshfs` still `Permission denied` — Docker’s default seccomp profile blocks the `mount` syscall |
| `-v ~/.ssh/edison_agent:…` | Agent has no key for remote SSH |
| host `ssh-config` / `known_hosts` / `state/*` | login and SSH config died with the container (closed 2026-10-09) |

**Volume-mount gotcha:** if the host path does not exist, Docker creates an empty **directory** at that path and mounts it. Inside the container `ls -la` then shows `drwxr-xr-x` instead of a key file. Always `ls` the host file before `docker run`.

---

## ssh-config (host file, bind-mounted)

`~/cursor-agent/ssh-config` → `/root/.ssh/config` (ro):

```
Host osgar.datahub.foxcraft.digital
    User edison
    Port 2201
    IdentityFile /root/.ssh/id_ed25519

Host dominikliss.foxcraft.digital
    User edison
    Port 2204
    IdentityFile /root/.ssh/id_ed25519

Host *
    UpdateHostKeys no
```

`UpdateHostKeys no` is required because ssh cannot rewrite the single bind-mounted `known_hosts` file (`Invalid cross-device link`).

---

## State folders (host)

| Host path | Mounted at | Purpose |
|-----------|------------|---------|
| `~/cursor-agent/ssh-config` | `/root/.ssh/config` (ro) | SSH hosts |
| `~/cursor-agent/known_hosts` | `/root/.ssh/known_hosts` | known_hosts |
| `~/cursor-agent/state/cursor` | `/root/.cursor` | Cursor CLI state |
| `~/cursor-agent/state/config-cursor` | `/root/.config/cursor` | Cursor login (`auth.json` — secret, do not commit) |

Persistence gap from the old docs is **closed** (2026-10-09).

---

## systemd

`/etc/systemd/system/cursor-agent.service` — enabled. Recreates the container and remounts sshfs at every boot.

- `Type=oneshot`
- `RemainAfterExit`
- `After=` / `Wants=` `network-online`
- `Requires=docker.service`
- `ExecStart=/root/cursor-agent/run.sh`

Reboot test passed (2026-10-09): container up, Cursor login kept, mount present.

---

## Agent SSH key (Edison → remotes)

A **new** key was generated on the new host 2026-10-09. The old key is retired.

```bash
ssh-keygen -t ed25519 -C "edison-agent" -f ~/.ssh/edison_agent
```

- Host filenames: `~/.ssh/edison_agent` / `edison_agent.pub` (ed25519, comment `edison-agent`, no passphrase)
- Inside the container the same files are mounted at `/root/.ssh/id_ed25519` and `.pub` — names differ on purpose
- New pubkey replaced the old `edison-agent` line in the shared `authorized_keys` on Stella (osgar port 2201 and dominikliss port 2204); both `-ssh` containers rebuilt. See [`../stella-server/dev-ssh-access.md`](../stella-server/dev-ssh-access.md)

---

## Working model: sshfs, not a local checkout

The agent does **not** clone the repo onto Edison. It works live on the remote tree, same idea as Cursor IDE Remote-SSH.

This is a live mount, not a copy. There is no second tree on Edison. Tools (`ls`, `grep -r`, `cursor-agent`) see a normal directory.

Further remotes: another `/mnt/<full-hostname>`. Same key, same network, new SSH config host. **TODO (Dominik):** add the `dominikliss` mount.

---

## Shared working tree warning

The sshfs mount points at the **live** working tree of osgar-datahub (`src/` bind-mounted into `osgar-datahub-ssh` and the `-dev` container). On 2026-10-09 the repo was on `main`, 2 commits ahead of origin, with 14+ modified files (uncommitted work by Dominik/Pawel).

**Rule:** the agent must **not** run `git checkout`, `switch`, `stash`, `reset` or `commit` in this tree. Edits go directly onto the running staging app (`dotnet watch` reloads). For real tasks a separate git worktree or clone is needed. **TODO (Dominik):** design that (not done yet).

Builds / tests / git should run **on Stella** via `ssh osgar.datahub.foxcraft.digital 'cd app && ...'` as user `edison` inside `osgar-datahub-ssh` (dotnet SDK, Node and supervisorctl are there), not inside the Edison container (no SDK there).

`git` needed `git config --global --add safe.directory /home/edison/app` for user `edison` inside `osgar-datahub-ssh` (repo owned `root:ubuntu`). Applied manually; it lives in the container’s `/home/edison/.gitconfig` and is **lost on rebuild**. **TODO (Dominik):** bake into `Dockerfile.ssh`.

Do not write the supervisorctl password into any doc. **TODO (Dominik):** `Dockerfile.ssh` for osgar contains it in plain text; move it to an env/secret and rotate.

---

## Headless usage

Cursor CLI is logged in with Dominik's Cursor account (`cursor-agent login`; do not write the email into the docs).

```bash
docker exec -w /mnt/osgar.datahub.foxcraft.digital cursor-agent \
  timeout 120 cursor-agent -p --trust --output-format text "<prompt>"
```

- `--trust` is required for a new directory in headless mode (otherwise `Workspace Trust Required`).
- No output appears until the run finishes (the first run looked like a hang).
- Timing: a question that scans the whole project over sshfs took ~23 s.
- `--force` lets print mode write files. A write test created a file on Stella through the mount.
- File permissions: default umask `022` produced files on Stella as `rw-r--r--` with ACL mask `r--`, so group 1000 (Dominik, Pawel) could **not** edit them. With umask `002` the mask is `rw-` and group members can edit. Use **`cursor-agent-run`** instead of `cursor-agent` for any run that may write files.
- Default model: **UNKNOWN**. **TODO (Dominik):** check with `cursor-agent models` / `--help`, and set `--model` explicitly for reproducible runs.

Not verified, from third-party sources: plan/ask mode may not be reliably read-only; max 3 simultaneous headless processes; unattended runs should use an API key (`CURSOR_API_KEY`), not set up yet.

Print / stream flags for the future web app: `-p --output-format stream-json`, `--resume <id>`.

---

## Cursor CLI — first-run notes (interactive)

- First start in a new directory: **Workspace Trust Required** — answer `a` (Trust this workspace), or pass `--trust` in headless mode.
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

When the planned web app starts runs (instead of a person at a TTY), add logging/audit of what the agent does on staging. See [README.md](README.md) for the adapter-layer decision.

---

## Adding another agent (checklist)

1. `~/<tool>-agent/Dockerfile` + image `<tool>-agent-image`.
2. `docker run --name <tool>-agent --network edison-net` with the same FUSE/seccomp flags if that tool also uses `sshfs`.
3. Reuse `/mnt/<full-hostname>` mounts (or remount in the new container). Reuse `edison_agent` unless a tool needs its own key.
4. Do not name the container `edison-*`.
