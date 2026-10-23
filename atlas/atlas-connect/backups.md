# Atlas Connect — backups

Atlas starts and drives every backup. The WordPress site never phones home and never stores S3 keys.

Index and vault fields: [`README.md`](README.md). Login: [`login.md`](login.md). Infection scan: [`security-scan.md`](security-scan.md).

---

## Policy

- Only Atlas can start a backup. WP admin is history + log retention.
- Every route is **`POST`** with a JSON body, including status. `secret` is never read from the query string.
- Send S3 credentials on **start, chunk, and cancel**. The plugin uses them for that request and does not persist them.
- The S3 destination is **pinned** on the site: bucket `atlas-wp-sites-backups`, Hetzner Object Storage over HTTPS. See [S3 destination is pinned](#s3-destination-is-pinned).
- One running job per site. A second start is `409` until the first is `complete`, `error`, or `cancelled`, or the job TTL expires (default 24h).
- Drive `/backup/chunk` **sequentially**. Do not parallelize the same `job_id`.
- HTTP client timeout: **at least 60 seconds** per chunk. A chunk's budget is `chunk_time_limit` (40s), counted from the request start; on `cgi-fcgi` hosts (Apache + mod_fcgid) it is capped at `fcgid_chunk_time_limit` (25s) and in-flight uploads at `fcgid_max_inflight_mb` (64 MB). New uploads stop early enough for the ones in flight to finish, based on the upload throughput measured on earlier chunks. Uploads still running 8s after the budget are cut off and the job fails with `… chunk time limit reached …`.

---

## Start

```http
POST /wp-json/atlas-connect/v1/backup/start
Content-Type: application/json

{
    "secret": "{atlas_secret_key}",
    "backup_type": "full",
    "s3": {
        "endpoint": "https://fsn1.your-objectstorage.com",
        "bucket": "atlas-wp-sites-backups",
        "region": "fsn1",
        "access_key": "{s3_access_key}",
        "secret_key": "{s3_secret_key}"
    }
}
```

| Field | Required | Notes |
|---|---|---|
| `secret` | yes | Site secret. JSON body only |
| `backup_type` | no | `full` (default), `db`, or `files`. Anything else becomes `full` |
| `s3.endpoint` | yes | Must resolve to `https://` + an allowed host (see below). `https://` is prepended if no scheme is given; `http://` is **rejected** |
| `s3.bucket` | yes | Must be exactly `atlas-wp-sites-backups` (case-sensitive) |
| `s3.region` | no | Default `fsn1`. Used for SigV4 signing only — keep it matching the endpoint host |
| `s3.access_key` | yes | Trimmed only, never otherwise altered |
| `s3.secret_key` | yes | Trimmed only, never otherwise altered |

Flat aliases also work: `s3_endpoint`, `s3_bucket`, `s3_region`, `s3_access_key`, `s3_secret_key`. Prefer the nested `s3` object.

S3 fields are read from the **JSON/form body only**. Query-string `s3_*` is ignored.

### S3 destination is pinned

Atlas chooses the endpoint per request, so without a pin anyone holding a site secret (from an Atlas IP) could make the site send requests — including its files, encrypted, and the plaintext `manifest.json` — to any host, including internal ones (Docker network, cloud metadata). The plugin therefore validates the S3 block on **start, chunk, and cancel**, before any work:

| Check | Rule | Failure |
|---|---|---|
| Scheme | Endpoint must start with `https://` (after `https://` is prepended to a bare host) | `400` `{ "error": "S3 endpoint is not allowed." }` |
| Host | Host (lower-cased, from `wp_parse_url`) must be one of `config.php` → `backup.s3_endpoint_hosts`: `fsn1.your-objectstorage.com`, `nbg1.your-objectstorage.com`, `hel1.your-objectstorage.com` | `400` `{ "error": "S3 endpoint is not allowed." }` |
| Bucket | Must equal `config.php` → `backup.s3_bucket`: `atlas-wp-sites-backups` | `400` `{ "error": "S3 bucket is not allowed." }` |
| Normalisation | The endpoint is rebuilt as `https://{host}` — any path, port, query, or `user@` part Atlas sends is dropped | — |

Examples:

| `s3.endpoint` sent | Result |
|---|---|
| `https://fsn1.your-objectstorage.com` | OK |
| `fsn1.your-objectstorage.com` | OK (`https://` prepended) |
| `https://FSN1.your-objectstorage.com/` | OK (host lower-cased, trailing slash dropped) |
| `https://fsn1.your-objectstorage.com:8443/x` | OK, but sent to `https://fsn1.your-objectstorage.com` |
| `http://fsn1.your-objectstorage.com` | `400` endpoint not allowed |
| `https://s3.eu-central-1.amazonaws.com` | `400` endpoint not allowed |
| `https://fsn1.your-objectstorage.com@evil.example` | `400` (host is `evil.example`) |

Checks run in this order: required fields (`endpoint`, `bucket`, `access_key`, `secret_key` all present → else `400` `S3 credentials are required (endpoint, bucket, access_key, secret_key).`), then endpoint, then bucket.

Changing the bucket or adding a region means editing `config.php` in the plugin and releasing. Atlas should validate the same rules in its Settings vault so a bad value fails there, not on every site.

If the S3 settings in Atlas change **while a job is running** and no longer pass these checks, `chunk` and `cancel` both return `400` and the job stays `running` until its TTL (24h) expires and the daily cron marks it `error`. Validate before saving.

---

## Drive until done

```
status = POST /backup/start
while status.status == "running":
    status = POST /backup/chunk   # same secret + job_id + s3
```

Stop when `status` is `complete`, `error`, or `cancelled`.

**Failed chunks (Atlas):** a chunk that returns `500` or can't be reached (Atlas maps network errors and other 5xx to `502`) is retried. Atlas waits 15s, reads `/backup/status/{job_id}`, and sends the next chunk. The backup fails, and Atlas cancels the job on the site, after 4 failed chunks in a row, or right away if status reports `error` / `cancelled`. A `500` with no JSON `error` field means the request never reached the plugin's error handling. On `cgi-fcgi` hosts (Apache + mod_fcgid) this is usually `FcgidIOTimeout` (default 40s) killing a chunk that ran too long. Otherwise it is a PHP fatal error; the plugin then sets the job to `error` with `PHP fatal error in <phase> phase: …`, so the next status read stops the backup. Atlas shows the first 160 characters of the response body. 4xx responses are not retried.

```http
POST /wp-json/atlas-connect/v1/backup/chunk
Content-Type: application/json

{
    "secret": "{atlas_secret_key}",
    "job_id": "{job_id}",
    "s3": { "endpoint": "…", "bucket": "…", "region": "fsn1", "access_key": "…", "secret_key": "…" }
}
```

---

## Status (optional UI poll)

Does **not** process a chunk and does **not** need S3 credentials.

```http
POST /wp-json/atlas-connect/v1/backup/status/{job_id}
Content-Type: application/json

{
    "secret": "{atlas_secret_key}"
}
```

`job_id` stays in the URL path (`[a-zA-Z0-9]+`, 12 random chars) — it is not a secret. Response is the [job payload](#job-payload-start--chunk--status).

**Changed 2026-10-23:** this used to be `GET …/backup/status/{job_id}?secret=…`. `GET` now returns `404 rest_no_route`, and a query-string secret is ignored (`401`). The secret no longer appears in access logs.

---

## Cancel

```http
POST /wp-json/atlas-connect/v1/backup/cancel
Content-Type: application/json

{
    "secret": "{atlas_secret_key}",
    "job_id": "{job_id}",
    "s3": { "endpoint": "…", "bucket": "…", "region": "fsn1", "access_key": "…", "secret_key": "…" }
}
```

S3 is required so the site can abort an in-flight multipart upload. Success: `{ "job_id": "…", "status": "cancelled" }`.

---

## Job payload (start / chunk / status)

```json
{
    "job_id":         "AbCdEfGh1234",
    "status":         "running",
    "phase":          "files",
    "progress":       67,
    "db_done":        12,
    "db_total":       12,
    "current_table":  "",
    "files_offset":   410,
    "files_total":    880,
    "objects_count":  390,
    "bytes_uploaded": 14800321,
    "error":          ""
}
```

| Field | Meaning |
|---|---|
| `status` | `running` \| `complete` \| `error` \| `cancelled` |
| `phase` | `inventory` \| `db` \| `files` \| `verify` \| `done` |
| `progress` | 0–100, approximate |
| `error` | Set when `status` is `error` or `cancelled` |
| `current_table` | Only during `phase=db` |

Never includes S3 keys, the site secret, or the backup data key.

---

## HTTP errors

| Status | Body | When |
|---|---|---|
| 401 | `{ "code": "rest_forbidden", "message": "Sorry, you are not allowed to do that.", "data": { "status": 401 } }` | Auth failed (SSL, IP, or secret — see [`README.md`](README.md#authentication-every-plugin-route)) |
| 400 | `{ "error": "S3 credentials are required (endpoint, bucket, access_key, secret_key)." }` | Missing S3 field (start / chunk / cancel) |
| 400 | `{ "error": "S3 endpoint is not allowed." }` | Endpoint not `https://` or host not allowed |
| 400 | `{ "error": "S3 bucket is not allowed." }` | Bucket is not `atlas-wp-sites-backups` |
| 400 | `{ "error": "Missing job_id" }` | chunk / cancel without `job_id` |
| 400 | `{ "error": "Job is not running." }` | cancel on a finished job |
| 409 | `{ "error": "A backup is already running." }` | start while another job runs |
| 404 | `{ "error": "Job not found" }` | Unknown or expired `job_id` |
| 404 | `{ "code": "rest_no_route", … }` | Sent as `GET` (e.g. old status call), or plugin inactive |
| 500 | `{ "error": "The PHP sodium extension is required for encrypted backups." }` / `{ "error": "Backup public key is missing or invalid." }` | Site misconfigured |

Route errors are `{ "error": "…" }`. Auth and routing errors are WordPress's `{ "code", "message", "data" }`. Parse with `data.error ?? data.message`.

---

## Phases

| `backup_type` | Phases |
|---|---|
| `full` | `inventory` → `db` → `files` → `verify` → `done` |
| `db` | `db` → `verify` → `done` |
| `files` | `inventory` → `files` → `verify` → `done` |

On `verify` the site checks that every packed file lies inside an uploaded pack, that every site file has a plaintext fingerprint (`files[]`), uploads `manifest.json` (plaintext), then lists the prefix and checks every object exists on S3 with the expected size (and ETag where known).

---

## What lands in the bucket

Prefix (fixed at job start, UTC date):

```
backups/{hostname}/{YYYY-MM-DD}/{job_id}/
```

`hostname` is `sanitize_file_name` of `home_url` host.

```
backups/client.com/2026-10-01/AbCdEfGh1234/
├── manifest.json                 ← plaintext JSON
├── db/
│   └── wp_options/
│       └── 0000.sql.gz           ← encrypted (may be 0001, 0002, …)
├── packs/
│   ├── 000001.pack               ← many small files, each encrypted on its own
│   └── 000002.pack
└── files/
    ├── wp-config.php             ← encrypted (only if > 1 MB; smaller files live in packs/)
    ├── wp-content/…
    └── uploads/…                 ← relocated uploads use files/uploads/ or files/_other/
```

- DB parts: `{prefix}db/{table}/{NNNN}.sql.gz` — gzip SQL, then encrypted as one S3 object.
- Site files: each file has a logical key `{prefix}` + `files/…` relative to `ABSPATH`. Parent `wp-config.php` (above ABSPATH) → `files/_other/wp-config.php`. How it is stored depends on size:

| File size | Stored as |
|---|---|
| ≤ 1 MB (`pack_file_max_kb`) | Byte range inside `{prefix}packs/NNNNNN.pack`. **No object exists at the file's own key** — find it via `manifest.packed` |
| ≤ 8 MB (`small_file_max_mb`) | Its own object at its key, `FLAG_SINGLE` |
| > 8 MB | Its own object at its key, multipart, `FLAG_FRAMED` |

- Packs are up to 16 MB (smaller on low-memory hosts). Nothing is compressed — media is stored as-is, only encrypted.
- `manifest.json` is **not** encrypted.

### Manifest

```json
{
    "job_id": "AbCdEfGh1234",
    "site_url": "https://client.com",
    "wp_version": "6.6.2",
    "table_prefix": "wp_",
    "backup_type": "full",
    "db_done": ["wp_options", "wp_posts"],
    "files_total": 880,
    "objects_count": 890,
    "bytes_uploaded": 14800321,
    "objects": [
        { "key": "backups/client.com/2026-10-01/AbCdEfGh1234/db/wp_options/0000.sql.gz", "bytes": 1200, "etag": "abc" },
        { "key": "backups/client.com/2026-10-01/AbCdEfGh1234/packs/000001.pack", "bytes": 16777301, "etag": "def" }
    ],
    "packed": [
        {
            "key":    "backups/client.com/2026-10-01/AbCdEfGh1234/files/wp-includes/js/tw-sack.js",
            "pack":   "backups/client.com/2026-10-01/AbCdEfGh1234/packs/000001.pack",
            "offset": 123456,
            "length": 4210
        }
    ],
    "files": [
        {
            "key":    "backups/client.com/2026-10-01/AbCdEfGh1234/files/wp-includes/js/tw-sack.js",
            "sha256": "b94d27b9934d3e08a52e52d7da7dabfac484efe37a5380ee9088f7ace2efcde9",
            "size":   4164,
            "mtime":  1727791200
        }
    ],
    "pack_format": {
        "version": 1,
        "compression": "none",
        "entry": "Bytes [offset, offset + length) of the pack object are one standalone encrypted blob (FLAG_SINGLE), identical to an unpacked object; AD = entry key."
    },
    "files_format": {
        "version": 1,
        "hash": "sha256",
        "size": "plaintext bytes actually read for upload",
        "mtime": "unix timestamp from filemtime at upload time"
    },
    "crypto": {
        "wrap": "crypto_box_seal",
        "aead": "xchacha20poly1305_ietf",
        "wrapped_data_key": "{hex}"
    },
    "created_at": "2026-10-01T18:00:00+00:00"
}
```

`wrapped_data_key` is hex. Atlas unwraps it with the **private** X25519 key. The site wipes the plaintext data key when the job finishes.

Multipart objects may have an empty `etag` in `objects[]`; size is still checked.

- `objects[]` lists real S3 objects: DB parts, files > 1 MB, and packs.
- `packed[]` lists every site file stored inside a pack. `key` is the file's logical key (same form as an unpacked file); `pack` is the object it lives in; `offset`/`length` are bytes within that pack.
- `files[]` lists every **site file** (packed or unpacked) with a plaintext fingerprint. DB parts are not included. Missing on backups taken before this field existed — treat as unknown.
- A complete file list for restore is `objects[]` keys under `files/` **plus** `packed[].key`. Prefer `files[].key` when present.

### Comparing backups without decrypting

Ciphertext and `objects[].etag` always differ across jobs (per-job data key + random nonce). Use `files[]` instead.

Match files by the path after the job prefix (`files/wp-content/…`). Then:

| Compare | Meaning |
|---|---|
| `sha256` differs | Content changed |
| `sha256` matches | Same bytes — do not download or decrypt |
| key only in A | Removed (or skipped) in B |
| key only in B | Added in B |
| `size` / `mtime` | Hints only. Same-length edits keep `size`; `mtime` can be spoofed |

`files[]` is an index, not a tamper seal. Anyone who can write the bucket can change an object and the hash next to it. For integrity, keep a trusted copy of the manifest (or its hash) in Atlas when the job completes.

A `db` backup has an empty `files[]`. Database change detection still requires decrypting `db/…/*.sql.gz`.

---

## Decrypt (Atlas restore)

Magic is the four bytes `DLC1`. Do not rename it.

```
[ DLC1 ][ version:1 ][ flag:1 ][ payload… ]
```

| Constant | Value |
|---|---|
| Magic | `DLC1` (4 bytes) |
| Version | `1` |
| `FLAG_SINGLE` | `0` — one AEAD message (small objects) |
| `FLAG_FRAMED` | `1` — length-prefixed frames (multipart / large files) |
| Data key | 32 bytes (`XChaCha20-Poly1305 IETF`) |
| Nonce | 24 bytes, prepended to each ciphertext |
| Public key | 32 bytes (64 hex) in site `config.php` |

### Unwrap the data key

```
data_key = crypto_box_seal_open( hex2bin(manifest.crypto.wrapped_data_key), atlas_public, atlas_private )
```

libsodium: `sodium_crypto_box_seal_open`. Wrapped blob is 80 bytes (160 hex).

### `FLAG_SINGLE` (most DB parts, files ≤ 8 MB, and every packed entry)

Associated data **`ad` = the full S3 object key** (the `objects[].key` string, not URL-encoded). For a packed entry it is `packed[].key` — the file's own key, **not** the pack key.

```
header = blob[0:6]          # DLC1 + ver + 0
sealed = blob[6:]           # nonce(24) + ciphertext
plaintext = xchacha20poly1305_ietf_decrypt(sealed[24:], ad, sealed[0:24], data_key)
```

### `FLAG_FRAMED` (large files / multipart)

Concatenated S3 object:

```
DLC1 | ver | 1 | frame0 | frame1 | …
frame = uint32_be(len(sealed)) | sealed
sealed = nonce(24) | ciphertext
```

Associated data for frame index `i` (0-based):

```
ad = s3_key + "\0" + str(i)     # e.g. "backups/…/video.mp4\0" + "0"
```

Decrypt each frame with that `ad`, concatenate plaintexts.

### Packed files

Each entry in `manifest.packed` is a complete `FLAG_SINGLE` blob sitting at a byte range of its pack. There is no pack header, index, or compression — the pack is just the blobs concatenated.

```
blob = GET pack with "Range: bytes={offset}-{offset + length - 1}"
plaintext = decrypt FLAG_SINGLE(blob, ad = entry.key)
write plaintext to the path for entry.key
```

For a full restore it is usually cheaper to download each pack once and slice all its entries locally (group `packed[]` by `pack`). For a single file, a range GET is enough.

### After decrypt

- `db/…/*.sql.gz` → gunzip → SQL (`DROP`/`CREATE`/`INSERT`). Import with the manifest `table_prefix` in mind.
- `files/…` → write relative to the restore root (`files/wp-config.php` → `{root}/wp-config.php`).

---

## Suggested Atlas driver (TypeScript)

```ts
type S3Creds = {
  endpoint: string;
  bucket: string;
  region?: string;
  access_key: string;
  secret_key: string;
};

type Job = {
  job_id: string;
  status: 'running' | 'complete' | 'error' | 'cancelled';
  phase: string;
  progress: number;
  error: string;
};

async function atlasPost(siteUrl: string, path: string, body: object): Promise<any> {
  const res = await fetch(`${siteUrl.replace(/\/$/, '')}/wp-json/atlas-connect/v1${path}`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(body),
    signal: AbortSignal.timeout(90_000),
  });
  const data = await res.json().catch(() => ({}));
  // Route errors use `error`; auth (401) and routing (404 rest_no_route) errors use WP's `message`.
  if (!res.ok) throw new Error(data.error ?? data.message ?? `HTTP ${res.status}`);
  return data;
}

// Atlas-side guard: the site rejects anything else with 400.
const ALLOWED_S3_HOSTS = ['fsn1.your-objectstorage.com', 'nbg1.your-objectstorage.com', 'hel1.your-objectstorage.com'];
const ALLOWED_S3_BUCKET = 'atlas-wp-sites-backups';

export function assertS3Allowed(s3: S3Creds): void {
  const url = new URL(/^https?:\/\//i.test(s3.endpoint) ? s3.endpoint : `https://${s3.endpoint}`);
  if (url.protocol !== 'https:' || !ALLOWED_S3_HOSTS.includes(url.hostname.toLowerCase())) {
    throw new Error('S3 endpoint must be https:// on Hetzner Object Storage (fsn1/nbg1/hel1).');
  }
  if (s3.bucket !== ALLOWED_S3_BUCKET) {
    throw new Error(`S3 bucket must be ${ALLOWED_S3_BUCKET}.`);
  }
}

export async function backupStatus(siteUrl: string, secret: string, jobId: string): Promise<Job> {
  // POST, secret in the body — GET with ?secret= is gone.
  return atlasPost(siteUrl, `/backup/status/${encodeURIComponent(jobId)}`, { secret });
}

export async function runBackup(
  siteUrl: string,
  secret: string,
  s3: S3Creds,
  backupType: 'full' | 'db' | 'files' = 'full',
): Promise<Job> {
  assertS3Allowed(s3);
  let job: Job = await atlasPost(siteUrl, '/backup/start', {
    secret,
    backup_type: backupType,
    s3,
  });

  while (job.status === 'running') {
    job = await atlasPost(siteUrl, '/backup/chunk', {
      secret,
      job_id: job.job_id,
      s3,
    });
  }

  return job;
}
```

On `complete`, list/get `{prefix}manifest.json` from the bucket with **Atlas’s** S3 client (do not use the site). Index `files[]` for change detection. Unwrap `crypto.wrapped_data_key` and decrypt objects and packed entries as above.

---

## Plugin internals (not required to implement Atlas)

`config.php` modules: `autologin`, `backup`, and `security` on. The old `email` module was removed.

Job state: option `atlas_backup_job_{job_id}` (not autoloaded). Active id: `atlas_backup_active_job`. TTL default 86400s. Daily cron `atlas_cleanup_cron` drops expired jobs and scratch files. Cron cannot abort S3 multipart (it has no credentials).

**Scratch files** (file list, inventory queue, object catalog, pack index, file fingerprints, and the **unencrypted** gzip DB part before it is encrypted) live in `atlas_scratch_dir( 'atlas-backup-tmp' )`:

1. `config.php` → `scratch_dir` if set, else `sys_get_temp_dir()/atlas-connect-{md5(ABSPATH)[0:12]}/atlas-backup-tmp/`, created `0700`. Outside the web root, so the DB dump cannot be downloaded even when Caddy/nginx serve uploads directly (`.htaccess` does nothing there).
2. Only if that is not writable (e.g. `open_basedir`): `uploads/atlas-backup-tmp-{24 random chars}/` with a deny-all `.htaccess` + `index.php`. The random suffix (`atlas_scratch_suffix` option) is what protects it on non-Apache servers.

The old fixed `uploads/atlas-backup-tmp/` (and `uploads/atlas-scan-tmp/`) is deleted on upgrade to DB version `1.2`. A backup that was running during the upgrade loses its scratch files and ends as `error` — start a new one. The inventory skips `atlas-backup-tmp*` and `atlas-scan-tmp*` under uploads.

The per-job data key (hex) sits in the job option while the job runs and is wiped on `complete` / `error` / `cancelled`. Someone with database read access during that window could decrypt that one backup. Accepted trade-off of chunked jobs.

S3 pin: `S3UploadService::sanitize_credentials()` with `config.php` → `backup.s3_bucket` + `backup.s3_endpoint_hosts`.

WP admin: Connection (secret — hash only, shown once on Generate; SSL; Auto-login Username) and Backups (max logs setting + history from `{prefix}atlas_backup_log`, last 100 runs, AJAX refresh via the generic table component).

Public key on this site: `config.php` → `backup.public_key`. Must match Atlas’s private key.

Legacy: sites that ran `dl-connect` are migrated once (`atlas_legacy_migrated`). REST path is only `atlas-connect/v1`.
