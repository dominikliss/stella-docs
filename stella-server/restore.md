# Restore runbook — restic snapshot onto a fresh server

**Purpose:** rebuild Stella from the latest restic snapshot after a total loss. A copy of this runbook is also kept on the server at `/opt/services/backup/RESTORE.md` (itself backed up). Backup behaviour: [`backup.md`](backup.md).

**Needed off-server:** restic password, Storage Box access (`u660268`), Hetzner API token.

**UNVERIFIED: this runbook has not been run end to end on a fresh server.** **TODO (Dominik):** test on a throwaway Hetzner Cloud server.

---

## 1. New server — packages

```bash
curl -fsSL https://get.docker.com | sh
# also install restic and acl
```

## 2. Storage Box SSH key

Restore the SSH key for the Storage Box, or authorize a new key on the Storage Box.

## 3. Confirm the repo

```bash
export RESTIC_REPOSITORY=sftp:u660268@u660268.your-storagebox.de:23//home/restic-repo
export RESTIC_PASSWORD_FILE=/root/restic.password
restic snapshots
```

Do not write the password into this file. Put it in `/root/restic.password` (chmod 600) from the password manager.

## 4. Restore the latest snapshot

```bash
restic restore latest --target /tmp/r
```

## 5. Copy trees back

```bash
cp -a /tmp/r/opt/. /opt/
cp -a /tmp/r/root/.ssh/. /root/.ssh/
```

## 6. Docker volumes

For each directory in `/tmp/r/var/lib/docker/volumes/*/`:

```bash
docker volume create <name>
cp -a /tmp/r/var/lib/docker/volumes/<name>/_data/. /var/lib/docker/volumes/<name>/_data/
```

This includes `services_caddy-data` (certs) and the SSH host-key volumes.

## 7. Host config

Copy from `/tmp/r/hostroot/etc/*` — at least `ufw`, `fail2ban`, `docker`, `ssh`, `netplan`, `cron.d`, plus the `docker-user-firewall` unit and the script its Exec lines reference. Then:

```bash
systemctl daemon-reload
systemctl enable --now docker-user-firewall fail2ban
ufw reload
```

## 8. Recreate the `edge` network

```bash
docker network create --subnet 172.20.0.0/16 edge
```

**Verified 2026-10-10:** `edge` is `172.20.0.0/16`, gateway `172.20.0.1`; health-api uses that gateway for the stella-api check.

## 9. Start core services

```bash
cd /opt/services && docker compose up -d --build
```

## 10. Start each app

```bash
cd /opt/apps/<type>/<app>
docker compose [-f docker-compose.dev.yml] up -d --build
```

## 11. Reload databases

DBs start empty. MariaDB / MySQL:

```bash
gunzip -c dumps/<container>.sql.gz | docker exec -i <container> sh -c 'MYSQL_PWD="${MARIADB_ROOT_PASSWORD:-$MYSQL_ROOT_PASSWORD}" mariadb -uroot'
```

Use `mysql` instead of `mariadb` on MySQL images. MSSQL: restore the `.bak` from `/dumps` via sqlcmd.

## 12. Check the live system

Check Atlas Monitoring, certs and sites.

---

## Slow part

Ollama models (~172 GB) re-pull and Docker images rebuild. Models are the only intentional backup exclusion.

---

## Open items

- **UNVERIFIED: this runbook has not been run end to end on a fresh server.**
- **TODO (Dominik):** test on a throwaway Hetzner Cloud server.
