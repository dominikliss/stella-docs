# Server Backups — restic → Hetzner Storage Box

**Purpose:** nightly encrypted backup of Stella's infra configs, app data, and databases, with no per-service script changes needed when a new service is added — services opt in via Docker labels.

---

## Storage backend

**Hetzner Storage Box** (`stella-backups`, user `u660268`, `u660268.your-storagebox.de:23`), accessed via restic's native SFTP backend, SSH-key auth (reuses Stella's own `~/.ssh/id_ed25519`, same key already trusted elsewhere on the server — e.g. GitHub cloning).

**Why Storage Box over Object Storage:** restic's SFTP backend works directly with no S3 gateway config; Storage Box's own API (GA since mid-2025) supports native `snapshot_plan` for a second, independent point-in-time safety net beyond restic's own repo; flat per-TB pricing with no per-request costs.

**Repo:** `sftp:u660268@u660268.your-storagebox.de:23//home/restic-repo` (repo ID `08ce1838fd`)
**Encryption:** client-side AES-256 (restic default) — the Storage Box only ever sees encrypted blobs. The repo password lives at `/opt/services/backup/restic.password` (chmod 600) — **also stored off-server**. Losing this file makes the entire backup unrecoverable, even with full Storage Box access.

---

## Label-based service discovery

No static include-list to maintain. Any container with `backup.enable=true` is picked up automatically by the discovery script:

```yaml
some-service:
  labels:
    - "backup.enable=true"
    - "backup.volumes=/path/inside/container"       # comma-separated, resolved to host paths via docker inspect
    - "backup.predump=/path/to/script/inside/container"  # optional, run via docker exec before backup
```

Adding a new service to the backup scope = adding these labels to its compose definition. No script edits required.

**Currently labeled:**
- `osgar-datahub-db` — `backup.predump=/usr/local/bin/backup-mssql.sh` (dumps all user DBs via `sqlcmd BACKUP DATABASE ... WITH COMPRESSION` to `/dumps`, mounted host-side at `.../osgar.datahub.foxcraft.digital/dumps`), `backup.volumes=/dumps`
- `osgar-datahub-dev` — `backup.volumes=/store-server-excel,/store-server-excel-backup,/temp_api` (real data folders; `src/` deliberately excluded — git-clonable)

**Not labeled (deliberately):**
- `advoapp-dev` — `src/` only, no real data, git-clonable
- `ollama` — models volume excluded by design, re-pullable via `ollama pull` (same principle as pre-backup-system days)

**MSSQL note:** raw volume copy of a live MSSQL data directory is unreliable — the `backup-mssql.sh` predump script runs `sqlcmd` inside the container first to produce a consistent `.bak`, which is what actually gets backed up. The raw `osgar-datahub-mssql-data` volume itself is **not** in the backup scope.

---

## Static infra paths (always included, not label-driven)

Config/secrets that aren't per-container:
- `/opt/services/docker-compose.yml`
- `/opt/services/caddy/Caddyfile`
- `/opt/services/deploy-api/allowed_signers`
- `/opt/services/deploy-api/logs`
- `/opt/services/backup/.env`
- Every `/opt/apps/*/.env` and `/opt/apps/*/docker-compose*.yml` (found via `find -maxdepth 2`)
- Every `/opt/services/*/.env`

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
1. Builds an include list: static paths + `find` results for per-app configs + resolved host paths from every `backup.enable=true` container's `backup.volumes` label
2. Runs each container's `backup.predump` command first (if set) via `docker exec` — aborts the whole run if any predump fails (status written as `predump_failed`, no partial/inconsistent backup taken)
3. `restic backup --files-from <include-list> --exclude ollama-models`
4. `restic forget --keep-daily 7 --keep-weekly 4 --keep-monthly 6 --prune`
5. Writes `/opt/services/backup/last-run.json` — `{"status": "success"|"failed", "duration_seconds": N, "timestamp": "..."}`

**No third-party monitoring service** (deliberately — avoided Healthchecks.io per preference). `health-api` reads `last-run.json` directly instead — see below.

---

## Monitoring — via health-api, not a third party

`health-api` mounts `/opt/services/backup:/opt/services/backup:ro` and exposes a `check_backup_status()` in its `/system/health` response:
- `ok` — last run succeeded, within 26h (nightly + 2h grace)
- `stale` — last successful run older than 26h (backup job stopped running, even if it hasn't explicitly failed)
- `failed` — last run's status wasn't `success` (predump or restic failure)
- `unknown` — no `last-run.json` yet (first run hasn't happened)

Atlas polls this the same way it already polls every other `health-api` field — one more card on the existing dashboard.

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
