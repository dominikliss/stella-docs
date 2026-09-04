# osgar-datahub — Dev Environment: Permissions, Build Tooling, and Live Reload

**Status:** Implemented 2026-09-03. Supersedes the original `osgar-datahub-ssh` setup in [`dev-ssh-access.md`](dev-ssh-access.md) with three fixes and one architectural addition (supervisord, later extended with a dedicated SCSS watcher process).

**Related**

- Original per-app SSH container pattern: [`dev-ssh-access.md`](dev-ssh-access.md)
- `.NET` app / `-dev` container pattern: [`dotnet-app-deployment.md`](dotnet-app-deployment.md)

---

## Problem 1 — `dotnet build` failing with MSB3374

**Symptom:** `MSB3374: The last access/last write time on file "...AssemblyInfo.cache" cannot be set. Access is denied.`

**Root cause:** the original permission fix in `dev-ssh-access.md` (`chgrp -R 1000` + `chmod -R g+rwX` + setgid on directories) was a one-time snapshot. Setgid forces new files to inherit **group ownership** but not **write bits**. Every time `osgar-datahub-dev` (running as root) regenerates `obj/`, new files land as `root:1000` mode `644` — group-readable but not group-writable. `dominik`/`pawel` (non-root, group members) can read but not write/set timestamps on files not explicitly `g+w`.

**One-off fix, applied to existing `obj`/`bin`:**
```bash
sudo chgrp -R 1000 /opt/apps/dotnet/osgar.datahub.foxcraft.digital/src/obj /opt/apps/dotnet/osgar.datahub.foxcraft.digital/src/bin
sudo chmod -R g+rwX /opt/apps/dotnet/osgar.datahub.foxcraft.digital/src/obj /opt/apps/dotnet/osgar.datahub.foxcraft.digital/src/bin
sudo find /opt/apps/dotnet/osgar.datahub.foxcraft.digital/src/obj /opt/apps/dotnet/osgar.datahub.foxcraft.digital/src/bin -type d -exec chmod g+s {} \;
```

**Permanent fix — default ACLs**, so files created later by root (`osgar-datahub-dev` rebuilds) automatically inherit group-write, not just group-ownership:
```bash
sudo apt-get install -y acl
sudo setfacl -R -m g:1000:rwX /opt/apps/dotnet/osgar.datahub.foxcraft.digital/src
sudo setfacl -R -d -m g:1000:rwX /opt/apps/dotnet/osgar.datahub.foxcraft.digital/src
```

**Caveat:** default ACLs are blanket — any new file dropped into `src/` (including something like `appsettings.Production.json` with real credentials) becomes automatically group-writable/readable to both `dominik` and `pawel`. Not a concern for this app (dummy dev DB), but worth checking before applying the same pattern to an app with real secrets in-tree.

**Apply this same fix to any future `-ssh` container** hitting the same class of error — this is a general gap in the original `dev-ssh-access.md` pattern, not specific to osgar-datahub.

---

## Problem 2 — Node/npm missing in `osgar-datahub-ssh`

**Symptom:** SCSS compilation (`node_modules/.bin/sass`) fails, `node`/`npm` not found.

**Root cause:** `Dockerfile.ssh`'s template in `dev-ssh-access.md` only installs `openssh-server curl wget ca-certificates git` — no Node, since the original design didn't anticipate SCSS build steps running inside the SSH container.

**Fix — add to `Dockerfile.ssh`'s apt block:**
```dockerfile
    && curl -fsSL https://deb.nodesource.com/setup_22.x | bash - \
    && apt-get install -y nodejs \
```

**Gotcha hit during setup:** the NodeSource install command was accidentally run directly in Stella's host shell instead of inside the Dockerfile, installing Node system-wide on the host by mistake. Cleaned up with:
```bash
sudo apt-get purge -y nodejs nodejs-doc libnode109 node-acorn node-busboy node-cjs-module-lexer node-undici node-xtend
sudo apt-get autoremove -y
```
**Lesson:** always double-check the shell prompt (`root@stella` vs a container prompt) before pasting multi-line install blocks — `docker compose build --no-cache` is the only reliable way to confirm a Dockerfile change actually took effect; a `CACHED` layer in the build output means nothing changed.

---

## Problem 3 — SSH host keys regenerating on every `Dockerfile.ssh` rebuild

**Symptom:** every rebuild of a `-ssh` container triggers "REMOTE HOST IDENTIFICATION HAS CHANGED" for every client that previously connected, requiring `ssh-keygen -R` before reconnecting.

**Root cause:** sshd's host keys are generated fresh inside the image on first boot and baked into the container filesystem — not persisted anywhere, so every rebuild produces new keys.

**Fix — persist `/etc/ssh` in a named volume**, applied to both `osgar-datahub-ssh` and `advoapp-ssh`:
```yaml
  osgar-datahub-ssh:
    volumes:
      - ./src:/home/dominik/app
      - ./src:/home/pawel/app
      - osgar-datahub-ssh-keys:/etc/ssh   # new
    ...

volumes:
  osgar-datahub-ssh-keys:   # new, top-level
```
(same pattern, `advoapp-ssh-keys`, for `advoapp-ssh`)

Once applied, `docker compose up -d` generates keys into the empty volume on first start; every subsequent rebuild reuses them. **Add this volume line to every future `-ssh` container by default** — it's a permanent gap in the original template, not an osgar-datahub-specific fix.

Clients with a stale `known_hosts` entry from before this fix, run once:
```bash
ssh-keygen -R "[95.217.144.93]:2201"   # osgar-datahub
ssh-keygen -R "[95.217.144.93]:2202"   # advoapp
```

---

## Architectural change — supervisord replaces the raw restart loop

**Original design** (per `dotnet-app-deployment.md`): `osgar-datahub-dev`'s command was a plain shell loop:
```
while true; do dotnet run; echo restarting; sleep 1; done
```
This has no supervision — if `dotnet run` (or later, `dotnet watch`) silently dies or hangs, nothing notices or restarts it, and there was no way for a separate container/user to trigger a targeted restart without either Docker socket access (breaks isolation) or fragile PID-based approaches (PID namespace sharing + `pkill` string matching, or sudoers rules keyed to a PID file) — several of which were tried and abandoned during this session as unnecessarily fragile or over-permissioned for the actual need.

**New design:** `osgar-datahub-dev` runs `supervisord`, which manages `dotnet watch` as a supervised child process (`autorestart=true`, `startretries=3`) and exposes an authenticated control port (`9001`) that `osgar-datahub-ssh` can reach over the shared `default` network — no shared PID namespace, no Docker socket, no root/sudo grant needed on either side.

### `supervisord.conf`

Bind-mounted into `osgar-datahub-dev`, **not baked into the image** — editable without a rebuild:

```ini
[supervisord]
nodaemon=true
logfile=/dev/stdout
logfile_maxbytes=0

[inet_http_server]
port=*:9001
username=agent
password=<see .env or ask Dominik>

[rpcinterface:supervisor]
supervisor.rpcinterface_factory = supervisor.rpcinterface:make_main_rpcinterface

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

[program:scss]
command=npm run scss:watch
directory=/src
autostart=true
autorestart=true
stdout_logfile=/dev/stdout
stdout_logfile_maxbytes=0
redirect_stderr=true

[supervisorctl]
serverurl=http://127.0.0.1:9001
username=agent
password=<same as above>
```

**`[rpcinterface:supervisor]` is required, not optional** — `[inet_http_server]` alone only serves the web status UI; without the RPC interface block, `supervisorctl` connects but every control command fails with "did not recognize the supervisor namespace commands." Easy to miss, cost real debugging time.

**`--non-interactive` on `dotnet watch`** avoids it expecting a TTY for its interactive hot-key commands (`Ctrl+R` to force-restart, etc.) — irrelevant and potentially hang-prone under supervisord, which has no interactive terminal attached.

**Port 9001 is not published to the host** (no `ports:` entry in compose for it) — only reachable from sibling containers on the same Compose network (`osgar-datahub-ssh` can reach it as `osgar-datahub-dev:9001`, per the network isolation table in [`dev-ssh-access.md`](dev-ssh-access.md)).

### Critical: `stopasgroup` / `killasgroup`

`dotnet watch` forks a multi-process chain (`dotnet watch` → `dotnet-watch.dll` → `dotnet run --no-build` → the actual `OsgarDatahub` binary). Without `stopasgroup=true` / `killasgroup=true`, `supervisorctl restart app` only signals the direct child — the deeper processes become orphans that keep holding port 8080.

**Symptom if this is missing:** every `restart app` leaves behind a full orphaned process chain. After a few restarts, the new instance fails at startup with:

```
Unhandled exception. System.IO.IOException: Failed to bind to address http://[::]:8080: address already in use.
```

**Confirmed via live incident 2026-09-03:** five stacked orphaned `dotnet-watch` chains were found via `docker exec osgar-datahub-dev ps aux` after repeated restarts without these flags. Fixed by adding both flags. Sanity-check after any restart:

```bash
docker exec osgar-datahub-dev ps aux
```

Expect exactly one `dotnet watch` → `dotnet-watch.dll` → `dotnet run` → `OsgarDatahub` chain. More than one chain means orphaned processes are accumulating.

**Known cosmetic delay:** `dotnet watch` doesn't respond cleanly to `SIGTERM`. supervisord waits out its stop timeout before escalating to `SIGKILL`:

```
WARN killing 'app' (330) with SIGKILL
```

Adds a few seconds to every restart. Not currently considered worth tuning (`stopsignal=SIGKILL` would skip the wait but skips a clean shutdown attempt too).

### `reread`/`update` can silently no-op on attribute changes

After adding `stopasgroup`/`killasgroup` to the config and running:

```bash
docker exec osgar-datahub-dev supervisorctl -c /etc/supervisor/conf.d/app.conf reread
docker exec osgar-datahub-dev supervisorctl -c /etc/supervisor/conf.d/app.conf update
```

supervisord reported **"No config updates to processes"** — it did not recognise the new flags as a change requiring action, even though the file on disk was correct. `restart app` under this stale in-memory config still failed to kill the process group.

**Fix:** when changing process-level attributes like `stopasgroup`/`killasgroup` (as opposed to `command`, which `reread` reliably detects), don't trust `reread`/`update` — force a full reload instead:

```bash
docker restart osgar-datahub-dev
```

This is safe here (unlike `pkill` from the `-ssh` container) because it's a normal Docker operation run as root on the Stella host, not a signal sent to an unprivileged process it doesn't own.

### Dockerfile changes

**`Dockerfile.dev`** — `supervisor` added to apt install.

**`Dockerfile.ssh`** — `supervisor` added (for the `supervisorctl` binary) plus a per-user config file:
```dockerfile
RUN mkdir -p /home/dominik /home/pawel \
    && printf '[supervisorctl]\nserverurl=http://osgar-datahub-dev:9001\nusername=agent\npassword=<...>\n' > /home/dominik/.supervisorctl.conf \
    && printf '[supervisorctl]\nserverurl=http://osgar-datahub-dev:9001\nusername=agent\npassword=<...>\n' > /home/pawel/.supervisorctl.conf \
    && chown dominik:dominik /home/dominik/.supervisorctl.conf \
    && chown pawel:pawel /home/pawel/.supervisorctl.conf
```

**Must be named `.supervisorctl.conf`**, not `.supervisorrc` — `supervisorctl` doesn't read the latter by default; this cost a debugging round-trip.

### PID namespace sharing removed

`pid: "container:osgar-datahub-dev"` + matching `depends_on` was removed once supervisord's network-based control made it unnecessary. **Leaving it in caused a real outage**: force-recreating `osgar-datahub-dev` gave it a new PID 1, which killed `osgar-datahub-ssh` too (exit 137) since it shared that container's PID namespace. Any future `-ssh` container should default to **no PID/network coupling to its `-dev` sibling** unless there's a specific, currently-unmet need for it.

### No second app instance

`osgar-datahub-ssh` no longer needs its own copy of the DB connection string — that was a workaround from an earlier (abandoned) design where the agent ran its own standalone `dotnet run` instance on a separate port (`8081`). That produced two independently-running app instances with no way to keep them in sync, and directly caused "rebuild doesn't show up" confusion. **Do not reintroduce a second running instance** — there must be exactly one running app, one URL, one restart mechanism.

---

## Problem 4 — SCSS changes not reflected (raw unstyled HTML)

**Symptom:** page loads with no styling applied — raw HTML, no CSS — even after `dotnet watch` picked up other code changes fine.

**Root cause:** `OsgarDatahub.csproj` compiles SCSS via a `CompileScss` MSBuild target (`BeforeTargets="Build"`, runs `node_modules/.bin/sass Styles/app.scss wwwroot/css/app.css --style=compressed --no-source-map`). `dotnet watch`'s hot-reload path does not reliably run a full MSBuild pass on every reload — it applies IL/Razor deltas directly in many cases — so `BeforeTargets="Build"` frequently doesn't fire, and edited `.scss` files never get recompiled into `app.css`. `dotnet watch` also doesn't watch `.scss` files at all by default (outside the project's normal compile globs), so even a full rebuild isn't reliably triggered by an SCSS-only edit.

**Fix — a dedicated, permanently-running `sass --watch` process, independent of `dotnet watch` entirely.** The project already ships an `npm run scss:watch` script for this; supervisord runs it as its own supervised program (the `[program:scss]` block in `supervisord.conf` above). With this running, SCSS edits recompile within a second or two regardless of what `dotnet watch` is doing — no manual `npm run scss` step, no full app restart needed for style-only changes.

**Note on Blazor scoped CSS (`.razor.css` → `OsgarDatahub.styles.css`):** this is a separate Blazor SDK build step, not Sass, and was not part of this fix. If scoped-CSS changes ever show the same staleness symptom, it needs its own investigation — likely does require a real `dotnet build`/`dotnet watch` restart, since it's not something `sass --watch` can produce.

---

## Problem 5 — white screen after a code change (build failure, not a crash)

**Symptom:** page goes completely white/blank. `supervisorctl status` still shows `app RUNNING` with a healthy uptime.

**Root cause:** `app RUNNING` only means the `dotnet watch` **process** is alive — it does not mean the last build succeeded. If an edit introduces a Razor/C# compile error, `dotnet watch` detects the failure, prints `Build FAILED` with the full error list, and then **holds** — it does not serve a broken build, and does not exit either, so supervisord sees a perfectly healthy long-running process the whole time. There is no crash for supervisord to catch or restart.

**Diagnosis — always check actual build output, not just process status:**
```bash
docker logs osgar-datahub-dev --tail 50
```
Look for `Build FAILED` and the specific `error RZ...` / `error CS...` lines — these are real compiler errors in the edited `.razor`/`.cs` file, not an infrastructure problem. `supervisorctl tail app` does not work for this — logs route to `docker logs` on the container, not a file `supervisorctl tail` can read.

**Fix is always in the code**, not the container. Once corrected and saved, `dotnet watch`'s polling file watcher picks up the change and rebuilds automatically — confirm with `docker logs` for `Build succeeded` / `Application started`.

---

## Migration workflow

Migrations in this app are **not** run via `dotnet ef database update`. The app calls `db.Database.Migrate()` in `Program.cs` on startup — so applying a schema change is really just "restart the app after the migration files exist in `src/`."

`dotnet-ef` is available in `osgar-datahub-ssh` (where migration files are authored by Cursor), deliberately **not** installed in `osgar-datahub-dev` (it's unused there — the running app only needs the runtime SDK).

1. Write migration files into `src/` from `osgar-datahub-ssh` via Cursor.
2. Restart the app so `Database.Migrate()` runs on startup:

   ```bash
   docker exec osgar-datahub-dev supervisorctl -c /etc/supervisor/conf.d/app.conf restart app
   ```

3. Confirm via logs:

   ```bash
   docker logs osgar-datahub-dev --tail 40
   ```

   Look for EF Core's `Applying migration '...'` line, or an exception if the migration fails against the current schema.

4. Sanity-check no orphaned processes:

   ```bash
   docker exec osgar-datahub-dev ps aux
   ```

   Expect exactly one `dotnet watch` → `dotnet-watch.dll` → `dotnet run` → `OsgarDatahub` chain.

**Not implemented / deliberately skipped:** there is no `[program:migrate]` supervisord block. This was considered and built during troubleshooting, then removed — unnecessary since migrations apply automatically via `Database.Migrate()` on every app start. Keep this in mind if this pattern is copied to a future app that does *not* auto-migrate on startup — that app would need a dedicated one-shot program instead.

---

## Day-to-day usage

**Normal edits:** `dotnet watch` inside `osgar-datahub-dev` hot-reloads automatically within a few seconds of a file change in the shared `src/` bind mount — no action needed from `osgar-datahub-ssh`.

**If a change doesn't show up after ~15 seconds**, from `osgar-datahub-ssh`:
```bash
supervisorctl -c ~/.supervisorctl.conf status
```
If not `app RUNNING`:
```bash
supervisorctl -c ~/.supervisorctl.conf restart app
```
Then confirm:
```bash
curl -s -o /dev/null -w "%{http_code}\n" https://osgar.datahub.foxcraft.digital/
```

**Cursor rule** (`~/app/.cursor/rules/rebuild-and-run.mdc`, committed to the app's own repo) encodes this fallback and explicitly forbids `pkill`, `sudo`, or Docker commands from that container — all three were tried and abandoned as more fragile or more permissive than necessary.

---

## Open follow-ups

- [ ] Same `Dockerfile.ssh` gaps (missing Node, no persistent host keys) likely apply to `advoapp-ssh` too — host key fix already applied there; Node and supervisord have not been evaluated for that app.
- [ ] `supervisorctl` credentials are currently a shared plaintext password baked into both Dockerfiles' build args — fine for this dummy-DB dev environment, but should not be copied as-is to any app with production data in `src/`.
- [ ] `DataProtection` key-ring warning (`No XML encryptor configured... may be persisted to storage in unencrypted form`) appears on every fresh run — cosmetic for dev, but worth a real fix if this pattern is ever used for anything closer to production.

**2026-09-04 additions:**
- `osgar-datahub-ssh` now has `sqlcmd` (mssql-tools18) for direct DB inspection — see [`osgar-datahub-ssh-sqlcmd.md`](osgar-datahub-ssh-sqlcmd.md).
- App logs are now written to a shared `app-logs/` bind mount so the SSH container can read them without Docker access — see [`osgar-datahub-ssh-app-logs.md`](osgar-datahub-ssh-app-logs.md).
