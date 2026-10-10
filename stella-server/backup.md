# Server Backups — restic → Hetzner Storage Box

**Purpose:** nightly encrypted backup of Stella's infra configs, app trees, Docker volumes, databases, and host config. Scope is automatic — a new service under `/opt/services` or `/opt/apps` is included without labels. Opt out with `backup.exclude=true`. Disaster-recovery runbook: [`restore.md`](restore.md) (also kept on the server at `/opt/services/backup/RESTORE.md`, which is itself backed up).

---

## Storage backend

**Hetzner Storage Box** (`stella-backups`, user `u660268`, `u660268.your-storagebox.de:23`), accessed via restic's native SFTP backend, SSH-key auth (reuses Stella's own `~/.ssh/id_ed25519`, same key already trusted elsewhere on the server — e.g. GitHub cloning).

**Why Storage Box over Object Storage:** restic's SFTP backend works directly with no S3 gateway config; Storage Box's own API (GA since mid-2025) supports native `snapshot_plan` for a second, independent point-in-time safety net beyond restic's own repo; flat per-TB pricing with no per-request costs.

**Repo:** `sftp:u660268@u660268.your-storagebox.de:23//home/restic-repo` (repo ID `08ce1838fd`)
**Encryption:** client-side AES-256 (restic default) — the Storage Box only ever sees encrypted blobs. The repo password lives at `/opt/services/backup/restic.password` (chmod 600) — **also stored off-server**. Losing this file makes the entire backup unrecoverable, even with full Storage Box access.

---

## Backup scope

Automatic and dynamic as of 2026-10-10. Labels are no longer the only way something gets included.

### Included automatically

The whole of `/opt/services` and `/opt/apps` (compose files, Dockerfiles, `.env`, `src/`, `html/`, `authorized_keys`, storage).

**Excluded via restic `--exclude`:** `ollama-models`, `*/logs`, `*/node_modules`, `*/__pycache__`, `*/.git`, `*/venv`, `/opt/apps/dotnet/*/src/bin`, `/opt/apps/dotnet/*/src/obj`.

### Docker volumes

Every volume used by a container is discovered via `docker inspect` and included, except `ollama-models` and volumes of DB containers that are dumped instead. This brings in `services_caddy-data` (certs, so no recert storm after a restore) and the SSH host-key volumes. Unused volumes are logged as `[warn] volume … not used by any container`. The three anonymous empty volumes found on 2026-10-10 were removed.

### Databases

Containers whose image matches mariadb/mysql/postgres are dumped each run to `/opt/services/backup/dumps/<container>.sql.gz` (`mariadb-dump` / `mysqldump --all-databases --single-transaction --routines --events --triggers`, root password read from the container's own env; `pg_dumpall` for Postgres). Their raw data volumes are then excluded.

| Dump outcome | Effect |
|---|---|
| Dump fails | Raw volume is backed up as a fallback; the run is marked `failed` with `reason: dump_failed` |
| Dump invalid after restic | Run marked `failed` with `reason: dump_invalid` |

After the restic run each dump is validated (non-empty, at least 1 KB, valid gzip, ends with `Dump completed`).

MSSQL (`osgar-datahub-db`) still uses the `backup.predump` label and `backup-mssql.sh`; its raw volume is excluded.

The `backup.enable`, `backup.volumes` and `backup.predump` labels keep working.

### Opt-out

Label `backup.exclude=true` on a container.

### Host config

All via `/hostroot` unless noted:

| Path | Notes |
|---|---|
| `/etc/ufw` | |
| `/etc/fail2ban` | |
| `/etc/cron.d` | |
| `/etc/docker` | |
| `/etc/ssh` | |
| `/etc/netplan` | |
| `/etc/network` | |
| `/etc/fstab` | |
| `/etc/hosts` | |
| `/var/spool/cron` | |
| `/root/.ssh` | Direct mount, not via `/hostroot` |
| `docker-user-firewall.service` unit | Plus the script its Exec lines reference |

### Not in any backup — keep off-server

restic password, Storage Box access, Hetzner API token, DNS records. **TODO (Dominik):** store them in the password manager.

**Models are the only intentional exclusion** from the on-server scope (`ollama-models`, ~172 GB, re-pullable).

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
    - /var/lib/docker/volumes:/var/lib/docker/volumes:ro
    - /:/hostroot:ro
  env_file:
    - ./backup/.env
  networks:
    - edge
  restart: unless-stopped
```

**Compose change 2026-10-10:** added mounts `/var/lib/docker/volumes:/var/lib/docker/volumes:ro` and `/:/hostroot:ro`.

**Dockerfile** (`/opt/services/backup-container/Dockerfile`): `debian:bookworm-slim` + `restic`, `docker.io` (CLI, to inspect/exec sibling containers via the mounted socket), **`openssh-client`** (restic's SFTP backend shells out to `ssh` — a slim image doesn't have this by default, hit and fixed during setup), `cron`.

**Gotcha hit during setup:** the container had no `known_hosts` entry for the Storage Box, and non-interactive restic can't answer the interactive host-key trust prompt (`Host key verification failed`). Fixed by mounting the host's `/root/.ssh` (which already has the Storage Box trusted from the initial manual test) into the container read-only — same mount pattern `deploy-api` already uses for its own SSH needs.

**Schedule:** `/etc/cron.d/backup-cron` inside the image — `0 3 * * * /opt/services/backup/run.sh`.

**Script:** `/opt/services/backup/run.sh` (bind-mounted via `/opt/services`, not baked into the image — lets it be edited without a rebuild):

1. Builds an include list: `/opt/services` + `/opt/apps` (entire trees, minus the excludes above) + every volume used by a container (minus `ollama-models` and dumped DB volumes) + host config via `/hostroot`
2. Dumps MariaDB/MySQL/Postgres containers; runs each `backup.predump` command (if set) via `docker exec` — a failed dump falls back to the raw volume and marks the run `failed` / `dump_failed`
3. `restic backup` with the exclude list above
4. Validates each dump (non-empty, ≥ 1 KB, valid gzip, ends with `Dump completed`) — a bad dump marks the run `failed` / `dump_invalid`
5. `restic forget --keep-daily 7 --keep-weekly 4 --keep-monthly 6 --prune`
6. Writes `/opt/services/backup/last-run.json` and appends one line to `/opt/services/backup/history.jsonl` (trimmed to the last 90 lines each run)
7. On Sundays, or when `last-verify.json` is missing, runs `restic check --read-data-subset=10%` and writes `/opt/services/backup/last-verify.json` (`status`, `timestamp`)

**`last-run.json` also contains** `snapshot_count`, `snapshot_bytes`, and a `dumps` array (`name`, `bytes`, `complete`). `history.jsonl` entries carry `snapshot_bytes`.

**Retention unchanged:** 7 daily, 4 weekly, 6 monthly.

**No third-party monitoring service** (deliberately — avoided Healthchecks.io per preference). `health-api` reads `last-run.json` / dump files / `history.jsonl` / `last-verify.json` directly instead — see below.

---

## Monitoring — via health-api, not a third party

`health-api` mounts `/opt/services/backup:/opt/services/backup:ro` and exposes two functions in its `/system/health` response.

**`backup` — current status:**

| `backup.status` | When |
|---|---|
| `ok` | Last run succeeded, within 26h, dumps present/fresh/`complete`, no shrink warning, weekly verify ok |
| `warning` | Snapshot bytes below 70% of the median of previous successful runs, or weekly verify older than 192h |
| `failed` | Last run was not success, a dump is missing/tiny/stale/`complete: false`, or weekly `restic check` failed |
| `stale` | Last successful run older than 26h |
| `unknown` | No `last-run.json` yet |

A `problems` list and a `verify` object are included in the response. Atlas displays both (2026-10-10). Atlas treats `warning` / `stale` as degraded and `failed` as down — see [`../atlas/monitoring.md`](../atlas/monitoring.md).

**`backup_history` — last 30 runs** (from `history.jsonl`): `get_backup_history()` reads the file, returns the 30 most recent entries in reverse-chronological order. Atlas dashboard tile "BACKUPS 30/30" counts these 30 entries, not a retention limit.

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

Full-server restore: [`restore.md`](restore.md).

---

## Known limits

- The shrink check uses logical snapshot size, so a deliberate large cleanup causes a one-off `warning`.
- The weekly check reads 10% of data per run.
- Everything between nightly runs (03:00) is lost on failure.

---

## Verified 2026-10-10

Full run OK (220s first run, 57s next), snapshot size about 3.2 GiB, repo about 818 MiB, `restic check` no errors, both dumps restored into fresh MariaDB 11 and MySQL 8.4 containers and showed the `wp` and `atlas_dev` databases.

Not yet verified: first unattended 03:00 run with the new script, and a full restore onto a fresh server.

---

## Open items

- First unattended 03:00 run with the new script — not yet observed.
- Full restore onto a fresh server — runbook written, **UNVERIFIED** end-to-end. **TODO (Dominik):** test on a throwaway Hetzner Cloud server. See [`restore.md`](restore.md).
- Off-server secrets (restic password, Storage Box access, Hetzner API token, DNS records). **TODO (Dominik):** store them in the password manager.
- Backup alert email — **UNVERIFIED** whether a failed backup sends an email via Brevo. **TODO (Dominik):** check whether the fake-failure test produced an email; if not, decide on an Atlas-side alert. See [`../atlas/monitoring.md`](../atlas/monitoring.md).
