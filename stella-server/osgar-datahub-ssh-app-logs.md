# osgar-datahub-ssh — App Logs via Shared `app-logs` Mount

**Status:** Implemented 2026-09-04. Extends [`osgar-datahub-dev-setup.md`](osgar-datahub-dev-setup.md) and [`osgar-datahub-ssh-sqlcmd.md`](osgar-datahub-ssh-sqlcmd.md) — same session.

**Why:** the Cursor agent in `osgar-datahub-ssh` needed a way to check build/runtime logs after triggering a restart (e.g. to confirm a migration applied, or to see a build error), without Docker access.

**Related**

- Container/network layout: [`dev-ssh-access.md`](dev-ssh-access.md)
- Permissions, supervisord, migration workflow: [`osgar-datahub-dev-setup.md`](osgar-datahub-dev-setup.md)
- `sqlcmd` DB access: [`osgar-datahub-ssh-sqlcmd.md`](osgar-datahub-ssh-sqlcmd.md)

---

## Why `supervisorctl tail` doesn't work

`supervisorctl tail` was the first thing tried and doesn't work:

```
dominik@...:~/app$ supervisorctl -c ~/.supervisorctl.conf tail app
app: ERROR (unknown error reading log)
```

Root cause: `/dev/stdout` inside a container is a pipe to the Docker logging driver, not a seekable file. supervisord's `tail`/`readProcessStdoutLog` RPC methods need to seek within the log file — a pipe can't do that, hence the generic error regardless of what's actually in the stream. This is a fundamental limitation of `stdout_logfile=/dev/stdout`, not something fixable with a flag.

**The `[inet_http_server]` / HTTP control channel cannot be used to read logs at all when `stdout_logfile` points at `/dev/stdout`** — it only works for process control (`start`/`stop`/`restart`/`status`). Worth remembering before assuming port 9001 is a complete substitute for `docker logs` access.

---

## Why not write the log into `/src`

First attempt: point `stdout_logfile` at a file inside `/src` (e.g. `/src/logs/app.log`). Works for reading, but `dotnet watch` polls the entire `/src` tree and treats the growing log file as a source change:

```
dotnet watch ⌚ File updated: ./logs/app.log
dotnet watch ⌚ No managed code changes to apply.
```

Harmless today (no actual rebuild triggered), but unnecessary noise and a latent risk if a future `dotnet watch` version or config reacts differently to non-code file changes inside its watched root. **Any log file the running app writes to must live outside `/src`.**

---

## Final design — shared `app-logs/` folder + forwarder process

A dedicated `app-logs/` folder at the project root (sibling to `src/`), mounted into both `osgar-datahub-dev` and `osgar-datahub-ssh`, plus a `tail -F` forwarder program in supervisord so `docker logs` keeps working unchanged.

### `docker-compose.dev.yml`

```yaml
  osgar-datahub-dev:
    volumes:
      - ./src:/src
      - ./app-logs:/app-logs
      # ...existing mounts unchanged

  osgar-datahub-ssh:
    volumes:
      - ./src:/home/dominik/app
      - ./src:/home/pawel/app
      - ./app-logs:/home/dominik/logs
      - ./app-logs:/home/pawel/logs
      # ...existing mounts unchanged
```

### `supervisord.conf`

```ini
[program:app]
command=dotnet watch run --urls http://+:8080 --non-interactive
directory=/src
autostart=true
autorestart=true
startretries=3
stopasgroup=true
killasgroup=true
stdout_logfile=/app-logs/app.log
stdout_logfile_maxbytes=10MB
stdout_logfile_backups=3
redirect_stderr=true

[program:app-log-forward]
command=tail -F /app-logs/app.log
autostart=true
autorestart=true
stdout_logfile=/dev/stdout
stdout_logfile_maxbytes=0
redirect_stderr=true
```

`app-log-forward` exists purely so `docker logs osgar-datahub-dev` keeps showing the same combined output it always has — nothing outside the container needs to know `stdout_logfile` changed.

**`stdout_logfile_maxbytes=10MB` / `stdout_logfile_backups=3` on `[program:app]`** — needed now that it's a real file rather than `/dev/stdout` (which never needed rotation, since Docker's own logging driver handled that). Without a cap, `app.log` grows unbounded for as long as `dotnet watch` runs.

---

## Usage — from `osgar-datahub-ssh`

```bash
tail -100 ~/logs/app.log
tail -f ~/logs/app.log
```

No `supervisorctl`, no Docker, no host access needed — plain file read over the bind mount.

**Confirmed working 2026-09-04:** after `--force-recreate` on both containers, `docker logs osgar-datahub-dev` showed the full expected build sequence (`Build succeeded`, `0 Warning(s)`, `0 Error(s)`) with no `File updated: ./logs/app.log` noise — confirming `dotnet watch` no longer sees the log file.

---

## If this pattern is copied to a future `.NET` app

- Add `app-logs/` to `.gitignore` (log content, not source).
- Do **not** add `app-logs` to the `backup.volumes` label — it's regenerable runtime output, not data worth restoring (same reasoning already applied to `src/` being excluded from backups in [`backup.md`](backup.md)).
- If the app's own log output volume is high, consider a shorter `stdout_logfile_backups` count, or route to a proper log file inside the app's own logging config rather than relying on supervisord's rotation.
