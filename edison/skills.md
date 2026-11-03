# Edison — Global agent skills

**Status:** Live on the Edison host **2026-11-03**. Host sync + `GET /v1/skills` / `POST /v1/skills/sync` verified inside `host-api`. Laravel `/` picker is in the Edison repo. `POST /v1/runs` has no `skills` field.

**Related:** [README.md](README.md) (role), [cursor-agent.md](cursor-agent.md) (`run.sh`, `/root/.cursor` bind-mount), [host-api.md](host-api.md) (`/v1/skills`, `/v1/skills/sync`), [app.md](app.md) (Laravel stays agent-agnostic).

---

## Decision

Global Foxcraft skills are authored only in [`foxcraftdigital/dev-docs`](https://github.com/foxcraftdigital/dev-docs) (`skills/*/SKILL.md` on `main`). Edison never edits them.

Cursor CLI discovers user-level skills from `~/.cursor/skills/` in headless `-p` runs. `cursor-agent` bind-mounts `~/cursor-agent/state/cursor` → `/root/.cursor`. The host keeps a shallow clone of `dev-docs`, copies `skills/` into that state dir, and refreshes it.

| Layer | v1 |
|---|---|
| GitHub `dev-docs` | Source of truth. Author / review / push here. |
| Edison **host** | Deploy key, shallow clone, `sync-skills.sh`, systemd timer + path unit, `run.sh` hook. |
| `host-api` | `GET /v1/skills`, `POST /v1/skills/sync`. Live 2026-11-03. |
| `POST /v1/runs` | **Unchanged.** No `skills` field. Picker inserts `/skill-name` into `prompt`. |
| Laravel / Edison UI | Composer `/` autocomplete + `+` search (`GET /api/skills`, `agent.skills`). **UNVERIFIED** on `edison.dev`. |

The web app never talks to GitHub and never shells out to Cursor. The host pulls; Cursor loads files from disk.

Do **not** write skills into an sshfs workspace. That dirties the live staging tree.

---

## Why the host, not Laravel

Laravel runs on Stella. A chat is `Browser → Laravel → POST /v1/runs → host-api → cursor-agent`.

Skills must exist **inside `cursor-agent`** at `/root/.cursor/skills/` before the CLI starts. `edison_agent` is the Stella sshfs key and cannot read GitHub. The box uses a separate read-only GitHub deploy key and a host-side pull.

`host-api` does not run `git`. The timer, `run.sh`, and (on demand) the `.path` unit call `sync-skills.sh`. host-api only writes a trigger file.

---

## Source of truth

| | |
|---|---|
| **Repo** | `git@github.com:foxcraftdigital/dev-docs.git` |
| **Branch** | `main` only. Fast-forward pulls. |
| **Path in repo** | `skills/<name>/SKILL.md` (+ optional extras) |
| **Authoring** | Edit in `dev-docs`, push. Laptops copy into `.cursor/skills/` via `git-push-with-tag`. Edison is a second consumer of the same tree. |
| **Pin** | Edison tracks `origin/main`, not a submodule SHA. Rollout `HEAD`: `cfe03f4`. Skills then: `git-push-with-tag`, `summarize-for-time-entry`. |
| **Clone** | `~/cursor-agent/dev-docs`, shallow (`--depth 1`). `git pull --ff-only origin main` works on that clone. |

Laravel Boost skills under a project’s `.claude/skills/` stay **per-repo**. Do not copy them into `/root/.cursor/skills/`.

---

## Layout on the Edison host

All under `~/cursor-agent/` (same family as `run.sh` and `state/`). Do not commit `~/cursor-agent` or keys.

| Host path | Purpose |
|---|---|
| `~/.ssh/id_ed25519` (+ `.pub`) | Default root key. Comment `edison-dev-docs`, no passphrase. Read-only deploy key on `foxcraftdigital/dev-docs`. **Not** `edison_dev_docs` (the design name was not used). **Not** `edison_agent`. |
| `~/cursor-agent/dev-docs/` | Shallow clone. Host-only. |
| `~/cursor-agent/sync-skills.sh` | Pull + exact copy + `status.json`. Idempotent. |
| `~/cursor-agent/state/cursor/skills/` | Exact copy of `dev-docs/skills/`. Seen as `/root/.cursor/skills/` in `cursor-agent`. Bind-mounted ro into host-api as `/etc/host-api/skills`. |
| `~/cursor-agent/skills-sync/` | `status.json` + `trigger`. Bind-mounted into host-api as `/var/lib/edison-skills-sync`. Whole directory (not a single file) so atomic replace keeps working. |
| `/etc/systemd/system/edison-skills-sync.service` | oneshot: `ExecStart=/root/cursor-agent/sync-skills.sh` |
| `/etc/systemd/system/edison-skills-sync.timer` | `OnBootSec=2min`, `OnCalendar=*:0/15`, `Persistent=true` |
| `/etc/systemd/system/edison-skills-sync.path` | `PathModified=…/skills-sync/trigger` → starts the service. Lets host-api request a sync. |

Existing mount (unchanged) from [cursor-agent.md](cursor-agent.md):

```
-v ~/cursor-agent/state/cursor:/root/.cursor
```

Cursor CLI then sees `/root/.cursor/skills/<name>/SKILL.md` on every workspace. No extra `docker run` flag. No `--skills` CLI flag.

---

## Auth

Uses the **default** root key `/root/.ssh/id_ed25519`. Comment `edison-dev-docs`. Added as a **read-only** deploy key on `foxcraftdigital/dev-docs`.

`sync-skills.sh` only sets `GIT_SSH_COMMAND='ssh -o BatchMode=yes'`. No `-i` / `IdentitiesOnly`. GitHub’s host key was accepted once (`ssh -T git@github.com`) into root’s `known_hosts`.

A GitHub deploy key is one repo only. A second repo needs its own key and an `~/.ssh/config` host alias.

`edison_agent` is unchanged (Stella sshfs only). Do not put a personal GitHub account key on Edison. Do not write the private key into docs.

**TODO (Dominik):** record the fingerprint (`ssh-keygen -lf ~/.ssh/id_ed25519.pub`). Do not invent it.

GitHub webhooks are out of scope: Caddy `/v1` is allowlisted to Stella + MacBook, not GitHub’s IPs. The box polls.

---

## Status file

`/root/cursor-agent/skills-sync/status.json` is written atomically (tmp + `mv`) on success **and** failure:

```json
{"ok":true,"commit":"<full sha>","finished_at":"2026-11-03T00:15:32Z","message":"ok"}
```

On failure: `ok` is `false`, `message` is the reason, `commit` is the current clone HEAD. The last good skills copy stays in place.

---

## Sync procedure (`sync-skills.sh`)

Live script at `/root/cursor-agent/sync-skills.sh`. Paths are hardcoded to `/root/...` (runs as root). Safe while a chat is in flight. A running CLI keeps the skills it already loaded; the **next** turn sees the new files.

1. `cd /root/cursor-agent/dev-docs` (missing → fail).
2. Dirty checkout (`git status --porcelain`) → fail, keep last good copy. Checked **before** the pull.
3. `GIT_SSH_COMMAND='ssh -o BatchMode=yes' git pull --ff-only -q origin main`. Fail → keep last good copy. Never reset, rebase, or commit on Edison.
4. Exact copy (no symlink) of `skills/*/SKILL.md` folders into `/root/cursor-agent/state/cursor/skills`. Stale dest folders are removed.
5. `diff -rq` source vs dest **unless** upstream `skills/` is missing.
6. `write_status true "ok"` or `write_status false "<reason>"`.

`run.sh` calls this **after** remount. Failure must **not** fail `cursor-agent.service`.

---

## When it runs

| Trigger | Interval | Why |
|---|---|---|
| `edison-skills-sync.timer` | every 15 min (`*:0/15`) + `OnBootSec=2min` | Normal update after a GitHub push. |
| `run.sh` | each `cursor-agent` recreate | Skills exist after remount. |
| `edison-skills-sync.path` | `trigger` file modified | `POST /v1/skills/sync` from host-api. |
| Manual | `systemctl start edison-skills-sync.service` or the script | Ops. |

Logs: `journalctl -u edison-skills-sync.service`.

Do **not** pull on every `POST /v1/runs`.

---

## What Cursor does

CLI version on Edison as of 2026-11-02: `2026.10.22-27dcf5f`. Skills load in headless `-p` from:

| Path in `cursor-agent` | Scope |
|---|---|
| `/root/.cursor/skills/` | **Global.** Every catalog workspace. |
| `{workspace}/.cursor/skills/` | Project-level copies already on a staging tree. Leave them alone. |

The agent applies a skill from its `description`, or when the prompt contains `/skill-name`.

---

## host-api

See [host-api.md](host-api.md#get-v1skills). Module `app/skills.py`. No extra Python dependency. Frontmatter `name` / `description` (including `>` / `|` blocks) parsed by hand.

| Method | Path | Result |
|---|---|---|
| GET | `/v1/skills` | `{ skills: [{ name, dir, description }], sync: <status.json or null> }` from the mounted copy. **503** if the skills mount is missing. |
| POST | `/v1/skills/sync` | Writes `trigger`, waits up to `EDISON_SKILLS_SYNC_TIMEOUT` (default 30 s) for `status.json` mtime to change. **200** same body as GET. **502** `{message: "Skills sync failed: …"}`. **202** still running. Concurrent calls are serialized. **503** if the sync-dir mount is missing. |

Laravel `GET /api/skills` reads `skills[].name` + `description` and ignores `dir` / `sync`. Missing or failed host call → `{ "skills": [] }`.

---

## Live-tree caveat

Edison runs `--force` on the **live** staging tree. Host `GUARD` text forbids `git checkout` / `switch` / `stash` / `reset` / `commit`.

`git-push-with-tag` is loaded globally. Its description triggers on "push", "deploy", "ship", "release" and tells the agent to commit, tag, and push. That conflicts with the guard. Decide: allowlist in `sync-skills.sh`, or mark the skill IDE-only in `dev-docs`. See [open-gaps](../open-gaps.md).

---

## Out of scope

- Authoring or editing skills on Edison
- Writing skills into sshfs mounts
- GitHub webhooks into `api.edison`
- Copying `.claude/skills` (Boost) into the global dir
- Changing `GUARD` text or the live-tree rule

---

## Tests (2026-11-02/03, on Edison)

| Test | Result |
|---|---|
| `ssh -T git@github.com` with the default key | authenticated as `foxcraftdigital/dev-docs` |
| Shallow clone | ok; `skills/` = `git-push-with-tag`, `summarize-for-time-entry` |
| `sync-skills.sh` by hand | `ok cfe03f4`; `docker exec cursor-agent ls /root/.cursor/skills` lists both |
| `systemctl start edison-skills-sync.service` | journal `sync-skills: ok cfe03f4`; timer next run on the quarter hour |
| `systemctl restart cursor-agent` | 5 mounts remounted (osgar, dominikliss, edison.dev, hypershape, atlas.dev), then `sync-skills: ok cfe03f4` |
| Trigger file write (`touch` on a new file) | `.path` fired, `finished_at` updated |
| `GET /v1/skills` inside host-api | both skills + descriptions + `sync` block |
| `POST /v1/skills/sync` inside host-api | **200** after a few seconds; `finished_at` 00:11:28 → 00:15:32 |

**UNVERIFIED:**

- `GET` / `POST /v1/skills` from Stella through Caddy (same bearer + allowlist).
- Push to `dev-docs` `main` → picked up within 15 min → a new Edison chat sees the changed description.
- `.path` firing on `touch` of an already existing `trigger` (host-api writes content, which does fire).
- Full box reboot (timer `OnBootSec` + `run.sh` sync).

---

## Errors

| Case | Behaviour |
|---|---|
| Deploy key missing / rejected | Sync exits non-zero. Last good copy stays. `status.json` `ok: false`. Timer retries. |
| `git pull --ff-only` cannot FF | Same. Do not rebase or `reset --hard` on Edison. |
| Dirty `~/cursor-agent/dev-docs` | Fail before pull. Edison must not have local skill edits. |
| Empty / missing upstream `skills/` | Dest cleaned; `diff -rq` skipped if source dir is missing. |
| `cursor-agent` not running | Host copy still updates. CLI sees it on next start via the bind-mount. |
| Chat in flight during copy | Current turn unchanged. Next `POST /v1/runs` uses new files. |
| host-api skills / sync-dir mount missing | **503** on `/v1/skills` or `/v1/skills/sync`. |
| `POST /v1/skills/sync` timeout | **202**. Sync may still finish; next GET shows the new `status.json`. |
