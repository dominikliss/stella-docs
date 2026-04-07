# IMAP sync service (Stella)

Small **Node.js + Express** helper on the Stella server that runs **`imapsync`** to copy mail from one IMAP account to another (full mailbox migration / consolidation). It is **not** the same as **ddashboard’s** IMAP import (`MailSyncService` / `MailChunkSyncService` into `dls_email`).

**Deployment**

- **Compose service:** `imap-sync` (build context `./imap-sync`) — see [`infrastructure.md`](infrastructure.md)
- **Listen port (container):** `3001`
- **Public entry:** Caddy on `:8080` under **`/imap-sync*`** → `imap-sync:3001`

**Caddy / path note:** Express registers routes at the **app root** (`/health`, `/start`, `/status/…`, `/jobs`, …). The reverse proxy must forward to the container so those paths hit the app (typically **strip** the `/imap-sync` prefix). Example: `http://<stella-host>:8080/imap-sync/start` → upstream `/start`.

**Related**

- Topology and ports: [`infrastructure.md`](infrastructure.md)
- ddashboard vs Stella responsibilities: [`../integration/ddashboard-and-stella-server.md`](../integration/ddashboard-and-stella-server.md)

---

## ddashboard proxy (WordPress)

The theme exposes logged-in **`dls/v1/imap-sync/*`** routes that forward to this service (`dls_imap_sync_base_url` — WordPress does not persist migration passwords). Proxied paths include **`/health`**, **`POST /start`**, **`GET /jobs`**, **`GET /status/{id}`**, **`GET /status/{id}/log`**, **`DELETE /jobs/{id}`**. The Werkzeuge UI uses **`GET /jobs`** for the live list and **`DELETE /jobs/{id}`** to cancel a running job.

---

## Credentials and `imapsync` invocation

- **`POST /start`** still accepts **`source.password`** and **`destination.password`** in JSON (HTTPS / trusted network only).
- The service writes each password to a **temporary file** (`mkdtemp` under OS tmpdir, file mode **0o600**) and passes **`--passfile1`** / **`--passfile2`** to **`imapsync`** (passwords are **not** passed on the command line).
- On process **`close`**, passfiles and temp dirs are **unlinked** via `cleanupPassfiles`.

Fixed **`imapsync`** flags (in addition to host, user, passfiles, port, ssl/tls): **`--all`**, **`--subscribeall`**, **`--nofoldersizes`**, **`--nolog`**.

---

## Validation (`POST /start`)

Request bodies are validated before spawn. **`400`** responses use JSON **`{ "error": "<message>" }`** (e.g. `source.host: required string`, `source.port: must be integer 1-65535`, `source.encryption: must be one of ssl, tls, none`). WordPress forwards these as **`400`** where possible so the UI can show the message.

---

## Job lifecycle and memory

| Mechanism | Detail |
|-----------|--------|
| **Storage** | In-memory `jobs[id]` — **lost on container restart** |
| **Log cap** | **`MAX_LOG_LINES` (500)** — oldest line dropped when appending (`pushLog`) |
| **Finished job TTL** | **`JOB_TTL_MS` (4 hours)** — every **10 minutes**, non-`running` jobs older than TTL are **deleted** from `jobs` |
| **Cancel** | **`DELETE /jobs/:id`** kills the child process; job **`status`** becomes **`cancelled`** |

---

## API

All endpoints use **JSON** where a body or structured response applies.

### `GET /health`

**Response** `200`

```json
{ "status": "ok", "running_jobs": 0 }
```

`running_jobs` counts jobs with `status === "running"`.

---

### `POST /start`

**Request body** — `source` and `destination` objects:

| Field | Type | Required | Notes |
|-------|------|----------|--------|
| `host` | string | yes | |
| `user` | string | yes | |
| `password` | string | yes | written to temp passfile only |
| `port` | integer | no | default **993** if omitted; must be **1–65535** if sent |
| `encryption` | string | no | **`ssl`** (default), **`tls`**, or **`none`** |

**Response** `200`: `{ "id": "<uuid>" }`

**Response** `400`: `{ "error": "<validation message>" }`

---

### `GET /status/:id`

Returns the job **without** the `log` array (summary only).

**Response** `404`: `{ "error": "Job not found" }`

---

### `GET /status/:id/log`

```json
{ "id": "<uuid>", "log": ["…", "…"] }
```

---

### `GET /jobs`

Object map **id → summary** (each entry omits `log`).

---

### `DELETE /jobs/:id`

Cancels a **running** job (`kill` on child). **`400`** if not running; **`404`** if unknown id.

**Response** `200`: `{ "cancelled": true }`

---

## Progress parsing (`imapsync` stdout)

Best-effort line parsing (adjust if **`imapsync`** output format changes):

| Signal | Regex / rule | Effect |
|--------|----------------|--------|
| Folder line | `^Folder\s+\d+/\d+\s+\[(.+?)\]` | Updates **`current_folder`**; previous folder appended to **`folders_done`** when it changes |
| Messages in folder | `^Host1: folder \[.+?\] selected (\d+) messages` | Adds to **`messages_total`** (running sum across folders) |
| Copied message | `^msg\s+.+?\s+\{\d+\}\s+copied to\s` | **`messages_transferred`** += 1; **`progress`** = round(transferred / total × 100) if total > 0 |
| Skipped | `^msg\s+.+?skipped` (case-insensitive) | **`messages_skipped`** += 1 |
| Failed count | `(\d+) messages? failed` (case-insensitive) | Sets **`messages_failed`** |

On exit: **`progress`** = **100** if exit code **0**, else keep last computed value; **`status`** = **`done`** or **`error`**; **`error`** message if non-zero exit.

---

## Job object (summary fields)

| Field | Notes |
|-------|--------|
| `status` | `running` \| `done` \| `error` \| `cancelled` |
| `started_at` / `finished_at` | ISO strings |
| `progress` | 0–100 |
| `current_folder` | string \| null |
| `folders_done` | string[] |
| `messages_transferred` / `messages_total` / `messages_skipped` / `messages_failed` | numbers |
| `error` | string \| null |
| `log` | full job only; omitted in `/status/:id` and `/jobs` summaries |

---

## Security and operations

1. **Passwords in HTTP JSON** to `POST /start` — use **TLS** and **network restriction** (Caddy, VPN, IP allowlist).
2. **Passfiles** live briefly on disk under the OS temp dir with **0o600**; cleaned on **`imapsync` exit**.
3. **No auth in the snippet** — same as before: protect at the edge.
4. **Single process** — job list is **one Node instance** only.

---

## Logs

Container stdout/stderr plus per-job **`log`** array (capped). Example:

```bash
docker logs <imap-sync-container-name> --tail 100
```

(Container name follows your compose project prefix — see [`infrastructure.md`](infrastructure.md).)
