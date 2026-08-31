# Server Backups — restic → Hetzner Storage Box

**Purpose:** nightly encrypted backup of Stella's infra configs, app data, and databases, with no per-service script changes needed when a new service is added — services opt in via Docker labels.

---

## Storage backend

**Hetzner Storage Box** (`stella-backups`, user `u660268`, `u660268.your-storagebox.de:23`), accessed via restic's native SFTP backend, SSH-key auth (reuses Stella's own `~/.ssh/id_ed25519`, same key already trusted elsewhere on the server — e.g. GitHub cloning).

**Why Storage Box over Object Storage:** restic's SFTP backend works directly with no S3 gateway config; Storage Box's own API (GA since mid-2025) supports native `snapshot_plan` for a second, independent point-in-time safety net beyond restic's own repo; flat per-TB pricing with no per-request costs.

**Repo:** `sftp:u660268@u660268.your-storagebox.de:23//home/restic-repo` (repo ID `08ce1838fd`)
**Encryption:** client-side AES-256 (restic default) — the Storage Box only ever sees encrypted blobs. The repo password lives at `/opt/services/backup/restic.password` (chmod 600) — **also stored off-server**. Losing this file makes the entire backup unrecoverable, even with full Storage Box access.

---

## Backup scope

### `/opt/services` — recursive, zero-maintenance

The entire `/opt/services` tree is backed up unconditionally. Every Dockerfile, `main.py`/`health.py`/etc. source file, `.env`, `allowed_signers`, and `docker-compose.yml` under `/opt/services` is covered the moment it exists on disk — no changes to `run.sh` needed when a new service is added here.

**Excluded via restic `--exclude`:** `ollama-models` (re-pullable), `*/logs`, `*/node_modules`, `*/__pycache__`, `*/.git`, `*/venv` (Python virtualenvs — fully reproducible via `pip install -r requirements.txt`).

### `/opt/apps` — label-driven, data only

Unlike `/opt/services`, only *data* under `/opt/apps` is backed up — not `src/` (git-clonable) or build artifacts. This is opt-in via `backup.enable=true` + `backup.volumes` labels on the relevant containers, since blanket-including all of `/opt/apps` would pull in every dev container's live source tree.

Any container with `backup.enable=true` is picked up automatically:

```yaml
some-service:
  labels:
    - "backup.enable=true"
    - "backup.volumes=/path/inside/container"       # comma-separated, resolved to host paths via docker inspect
    - "backup.predump=/path/to/script/inside/container"  # optional, run via docker exec before backup
```

Adding a new service to the backup scope = adding these labels to its compose definition. No script edits required.

**Currently labeled:**
- `osgar-datahub-db` — `backup.predump=/usr/local/bin/backup-mssql.sh` (dumps all user DBs via `sqlcmd BACKUP DATABASE ... WITH COMPRESSION`), `backup.volumes=/dumps`
- `osgar-datahub-dev` — `backup.volumes=/store-server-excel,/store-server-excel-backup,/temp_api` (real data folders; `src/` deliberately excluded — git-clonable)

**Not labeled (deliberately):** `advoapp-dev` (`src/` only, git-clonable), `ollama` (models excluded, re-pullable).

**MSSQL note:** raw volume copy of a live MSSQL data directory is unreliable — the `backup-mssql.sh` predump script runs `sqlcmd` inside the container first to produce a consistent `.bak`, which is what actually gets backed up. The raw `osgar-datahub-mssql-data` volume itself is **not** in the backup scope.

---

## The `backup` container

Runs on `edge`, no published port — same pattern as `deploy-api`/`health-api`. Cron-scheduled (not systemd — kept consistent with the rest of the stack being Docker Compose):

```yaml
backup:
  build: ./backup-container
  container_name: backup
  volumes:
    - /var/run/docker.sock:/var/run/docker.sock
    - /opt/apps:/opt/apps
    - /opt/services:/opt/services
    - /root/.ssh:/root/.ssh:ro
  env_file:
    - ./backup/.env
  networks:
    - edge
  restart: unless-stopped
```

**Dockerfile** (`/opt/services/backup-container/Dockerfile`): `debian:bookworm-slim` + `restic`, `docker.io` (CLI, to inspect/exec sibling containers via the mounted socket), **`openssh-client`** (restic's SFTP backend shells out to `ssh` — a slim image doesn't have this by default, hit and fixed during setup), `cron`.

**Gotcha hit during setup:** the container had no `known_hosts` entry for the Storage Box, and non-interactive restic can't answer the interactive host-key trust prompt (`Host key verification failed`). Fixed by mounting the host's `/root/.ssh` (which already has the Storage Box trusted from the initial manual test) into the container read-only — same mount pattern `deploy-api` already uses for its own SSH needs.

**Schedule:** `/etc/cron.d/backup-cron` inside the image — `0 3 * * * /opt/services/backup/run.sh`.

**Script:** `/opt/services/backup/run.sh` (bind-mounted via `/opt/services`, not baked into the image — lets it be edited without a rebuild):
1. Builds an include list: `/opt/services` (entire tree) + resolved host paths from every `backup.enable=true` container's `backup.volumes` label
2. Runs each container's `backup.predump` command first (if set) via `docker exec` — aborts the whole run if any predump fails (status written as `predump_failed`, no partial/inconsistent backup taken)
3. `restic backup --files-from <include-list> --exclude ollama-models --exclude '*/logs' --exclude '*/node_modules' --exclude '*/__pycache__' --exclude '*/.git' --exclude '*/venv'`
4. `restic forget --keep-daily 7 --keep-weekly 4 --keep-monthly 6 --prune`
5. Writes `/opt/services/backup/last-run.json` — `{"status": "success"|"failed", "duration_seconds": N, "timestamp": "..."}` — and **appends** the same object as one line to `/opt/services/backup/history.jsonl` (one JSON line per run, unbounded — pruning is health-api's concern)

**No third-party monitoring service** (deliberately — avoided Healthchecks.io per preference). `health-api` reads `last-run.json` directly instead — see below.

---

## Monitoring — via health-api, not a third party

`health-api` mounts `/opt/services/backup:/opt/services/backup:ro` and exposes two functions in its `/system/health` response:

**`backup` — current status** (from `last-run.json`):
- `ok` — last run succeeded, within 26h (nightly + 2h grace)
- `stale` — last successful run older than 26h (backup job stopped running, even if it hasn't explicitly failed)
- `failed` — last run's status wasn't `success` (predump or restic failure)
- `unknown` — no `last-run.json` yet (first run hasn't happened)

**`backup_history` — last 30 runs** (from `history.jsonl`): `get_backup_history()` reads the file, returns the 30 most recent entries in reverse-chronological order. Lets Atlas show a trend/history view rather than just current status. Each entry has the same shape as `last-run.json`: `{"status": "...", "duration_seconds": N, "timestamp": "..."}`.

Atlas polls both the same way it already polls every other `health-api` field.

---

## Manual operations

**Trigger a backup manually:**
```bash
docker exec backup /opt/services/backup/run.sh
```

**List snapshots:**
```bash
set -a && source /opt/services/backup/.env && set +a
restic snapshots
```

**Restore a specific path from the latest snapshot:**
```bash
restic restore latest --target /tmp/restore-test --include /opt/apps/dotnet/osgar.datahub.foxcraft.digital/dumps
```

**Check what's in a snapshot without restoring:**
```bash
restic ls latest
```
