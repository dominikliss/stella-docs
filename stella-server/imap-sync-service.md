# IMAP sync service (Stella)

Small **Node.js + Express** helper on the Stella server that runs **`imapsync`** to copy mail from one IMAP account to another (full mailbox migration / consolidation). It is **not** the same as **ddashboard’s** IMAP import (`MailSyncService` / `MailChunkSyncService` into `dls_email`).

**Deployment**

- **Compose service:** `imap-sync` (build context `./imap-sync`) — see [`infrastructure.md`](infrastructure.md)
- **Listen port (container):** `3001`
- **Public entry:** Caddy on `:8080` under **`/imap-sync*`** → `imap-sync:3001`

**Caddy / path note:** Express registers routes at the **app root** (`/start`, `/status/…`, `/jobs`, …). The reverse proxy must forward to the container so those paths hit the app (typically **strip** the `/imap-sync` prefix). Example public URLs: `http://<stella-host>:8080/imap-sync/start` → upstream `/start`.

**Related**

- Topology and ports: [`infrastructure.md`](infrastructure.md)
- ddashboard vs Stella responsibilities: [`../integration/ddashboard-and-stella-server.md`](../integration/ddashboard-and-stella-server.md)

---

## ddashboard proxy (WordPress)

The theme exposes logged-in **`dls/v1/imap-sync/*`** routes that forward to this service (`dls_imap_sync_base_url` — no credentials stored in WordPress). **`GET /jobs`** is proxied so the Werkzeuge UI can list migrations and progress without calling Stella from the browser.

---

## Dependencies

- **`imapsync`** available on `PATH` inside the container (install in Dockerfile image).
- Process spawned: `imapsync` with CLI flags derived from the JSON body (see below).

---

## API

All endpoints accept/return **JSON** (`Content-Type: application/json` where a body is used).

### `POST /start`

Starts a new job: spawns `imapsync` with source and destination credentials.

**Request body**

| Field | Type | Required | Notes |
|-------|------|----------|--------|
| `source` | object | yes | IMAP “from” account |
| `source.host` | string | yes | |
| `source.user` | string | yes | |
| `source.password` | string | yes | |
| `source.port` | number | no | default `993` |
| `source.encryption` | string | no | `"ssl"` (default), `"tls"`, or `"none"` |
| `destination` | object | yes | IMAP “to” account — same shape as `source` |

**`imapsync` flags** (fixed in addition to host/user/password/port):

- `--all`, `--subscribeall`, `--nofoldersizes`, `--nolog`
- Encryption: maps to `--ssl1` / `--tls1` / `--nossl1 --notls1` for source and `--ssl2` / … for destination.

**Response** `200`

```json
{ "id": "<uuid>" }
```

**Response** `400` — missing required fields

```json
{ "error": "Missing required fields" }
```

---

### `GET /status/:id`

Returns the job **without** the `log` array (summary only).

**Response** `200` — job object (shape below, minus `log`)

**Response** `404`

```json
{ "error": "Job not found" }
```

---

### `GET /status/:id/log`

Returns the full log lines for a job.

**Response** `200`

```json
{ "id": "<uuid>", "log": ["…", "…"] }
```

**Response** `404` — same as above

---

### `GET /jobs`

Map of **all** jobs (id → summary), each entry **omits** `log`.

**Response** `200`

```json
{
  "<uuid>": { "status": "running", "started_at": "…", … },
  "…": { … }
}
```

---

### `DELETE /jobs/:id`

Cancels a **running** job by killing the child process.

**Response** `200`

```json
{ "cancelled": true }
```

**Response** `404` — unknown id

**Response** `400` — job exists but `status !== "running"`

---

## Job object (in-memory)

Jobs are stored in a process-local object (`jobs[id]`). They are **lost on container restart**; there is no database persistence in this script.

| Field | Type | Notes |
|-------|------|--------|
| `status` | string | `"running"` \| `"done"` \| `"error"` \| `"cancelled"` |
| `started_at` | string (ISO) | |
| `finished_at` | string (ISO) \| `null` | set when the process exits or job is cancelled |
| `progress` | number | 0–100; from `messages_transferred / messages_total` while running; set to 100 on success exit |
| `current_folder` | string \| `null` | parsed from `imapsync` stdout (`Folder […]`) |
| `folders_done` | string[] | folders completed when moving to the next |
| `messages_transferred` | number | from `(\d+)/(\d+)` on stdout |
| `messages_total` | number | |
| `messages_skipped` | number | from `N messages skipped` (case-insensitive) |
| `messages_failed` | number | from `N messages failed` |
| `log` | string[] | stdout/stderr lines (only in full job; omitted in `/status/:id` and `/jobs` summaries) |
| `error` | string \| `null` | e.g. `imapsync exited with code <n>` when exit code ≠ 0 |

**Stdout parsing** (best-effort): folder lines, progress fraction, skipped/failed counts — depends on `imapsync` output format; adjust regexes if the binary’s log format changes.

---

## Security and operations

1. **Credentials in transit:** `POST /start` carries **plain passwords**. Only call over **HTTPS** (or VPN) and restrict who can reach `:8080` / the imap-sync path.
2. **No authentication in the snippet:** Put the service **behind** Caddy auth, IP allowlist, or VPN — same trust zone as other internal Stella tools.
3. **Not integrated with ddashboard:** WordPress does not call this API today; it is an **operator / migration** tool on Stella.
4. **Single-instance memory:** Listing jobs reflects **this process only**; scaling to multiple replicas would require shared storage and is out of scope for the current script.

---

## Logs

Use Docker logs for the compose service, e.g.:

```bash
docker logs <imap-sync-container-name> --tail 100
```

(Container name follows your compose project prefix — see [`infrastructure.md`](infrastructure.md).)
