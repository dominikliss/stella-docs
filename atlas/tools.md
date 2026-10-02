# Atlas tools — IMAP migration

Atlas **Tools → IMAP Migration** is the operator UI for Stella’s `imap-sync` service (Express + `imapsync`). It copies one mailbox onto another. It is **not** ddashboard’s IMAP import into MySQL.

Stella-side contract: [`../stella-server/imap-sync-service.md`](../stella-server/imap-sync-service.md).  
Why this left ddashboard: [`../integration/system-overview.md`](../integration/system-overview.md#6-what-moved-out-of-ddashboard-2026-09-02).

---

## Split of responsibility

| Layer | Does |
|-------|------|
| Atlas UI | Collect source/destination IMAP settings, test a single server, start/cancel, show history |
| Atlas API | Validate, call Stella, persist a history row **without passwords** |
| Atlas `ImapConnectionTestService` | Direct IMAP login test (does not go through Stella) |
| Stella `imap-sync` | Spawn `imapsync`, keep live job + log in memory |
| ddashboard mail | Unrelated — imports into `dls_mail_*` |

Passwords are sent to Stella over HTTPS and written to 0600 passfiles on Stella for the `imapsync` child. Atlas **never** stores them.

---

## Config

```
STELLA_IMAP_SYNC_URL=https://stella.foxcraft.digital/imap-sync
```

Caddy strips `/imap-sync` before Express. Atlas clients call `{base}/start`, `{base}/status/{id}`, `{base}/status/{id}/log`, `{base}/jobs/{id}`, `{base}/health`.

If the env var is empty, start/status/log/cancel return **503**.

There is **no** SSH signature on this path. Ingress is the same network restriction as other Stella HTTP services.

---

## Job lifecycle

```
POST /api/tools/imap-migration/start
  → Stella POST /start
  ← stella_job_id
  → ImapMigrationJob row (hosts, users, ports, encryption, initiated_by, status=running)

GET  /api/tools/imap-migration/jobs/{id}/status
  → Stella GET /status/{stella_job_id}
  → update progress / counts / current_folder / status

GET  /api/tools/imap-migration/jobs/{id}/log
  → Stella GET /status/{stella_job_id}/log   (not stored in Atlas)

DELETE /api/tools/imap-migration/jobs/{id}
  → Stella DELETE /jobs/{stella_job_id}
  → local status=cancelled
```

Stella status map: `done` / `error` / `cancelled` / else → `running`.

If Stella returns “job not found” (container restart or 4 h TTL), Atlas marks the row `error` with `Stella lost the job record (service may have restarted).`

---

## Atlas data model

`imap_migration_jobs`:

- `stella_job_id`, `initiated_by` (local user name/email)
- source/destination `host`, `user`, `port`, `encryption` — **no password columns**
- `status`, `progress`, message counts, `current_folder`, `error`
- `started_at`, `finished_at`

List endpoint returns the latest 100 rows.

---

## Connection test

`POST /api/tools/imap-migration/test-connection` uses `ImapConnectionTestService` from the Atlas host (PHP IMAP), so the operator can verify credentials **before** Stella starts a long copy. This does not create a Stella job.
