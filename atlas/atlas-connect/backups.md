# Atlas Connect — backups

Atlas starts and drives every backup. The WordPress site never phones home and never stores S3 keys.

Index and vault fields: [`README.md`](README.md). Login: [`login.md`](login.md).

---

## Policy

- Only Atlas can start a backup. WP admin is history + log retention.
- Send S3 credentials on **start, chunk, and cancel**. The plugin uses them for that request and does not persist them.
- One running job per site. A second start is `409` until the first is `complete`, `error`, or `cancelled`, or the job TTL expires (default 24h).
- Drive `/backup/chunk` **sequentially**. Do not parallelize the same `job_id`.
- HTTP client timeout: **at least 60 seconds** per chunk (the site aims for ~25s of work).

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
        "bucket": "atlas-backups",
        "region": "fsn1",
        "access_key": "{s3_access_key}",
        "secret_key": "{s3_secret_key}"
    }
}
```

| Field | Required | Notes |
|---|---|---|
| `secret` | yes | Site secret |
| `backup_type` | no | `full` (default), `db`, or `files`. Anything else becomes `full` |
| `s3.endpoint` | yes | `https://` is prepended if missing |
| `s3.bucket` | yes | |
| `s3.region` | no | Default `fsn1` |
| `s3.access_key` | yes | |
| `s3.secret_key` | yes | |

Flat aliases also work: `s3_endpoint`, `s3_bucket`, `s3_region`, `s3_access_key`, `s3_secret_key`. Prefer the nested `s3` object.

S3 fields are read from the **JSON/form body only**. Query-string `s3_*` is ignored.

---

## Drive until done

```
status = POST /backup/start
while status.status == "running":
    status = POST /backup/chunk   # same secret + job_id + s3
```

Stop when `status` is `complete`, `error`, or `cancelled`.

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
GET /wp-json/atlas-connect/v1/backup/status/{job_id}?secret={atlas_secret_key}
```

`job_id` is `[a-zA-Z0-9]+` (12 random chars).

`?secret=` will appear in the site’s access logs. Do not put S3 keys on this URL.

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

| Status | When |
|---|---|
| 401 | Auth failed (SSL, IP, or secret) |
| 400 | Missing `s3` / missing `job_id` / S3 not configured in the request / job not running (cancel) |
| 409 | `{ "error": "A backup is already running." }` |
| 404 | Unknown `job_id` |
| 500 | Sodium missing or `backup.public_key` invalid on the site |

Error body is always `{ "error": "…" }`.

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
  if (!res.ok) throw new Error(data.error ?? `HTTP ${res.status}`);
  return data;
}

export async function runBackup(
  siteUrl: string,
  secret: string,
  s3: S3Creds,
  backupType: 'full' | 'db' | 'files' = 'full',
): Promise<Job> {
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

`config.php` modules: `autologin` and `backup` on; `security` / `email` off.

Job state: option `atlas_backup_job_{job_id}` (not autoloaded). Active id: `atlas_backup_active_job`. TTL default 86400s. Daily cron `atlas_cleanup_cron` drops expired jobs and scratch files under `uploads/atlas-backup-tmp/` (file list, inventory queue, object catalog, pack index, file fingerprints, DB part). Cron cannot abort S3 multipart (it has no credentials).

WP admin: Connection (secret, SSL, default autologin user), Backups (history + max logs only), Log table `{prefix}atlas_backup_log`.

Public key on this site: `config.php` → `backup.public_key`. Must match Atlas’s private key.

Legacy: sites that ran `dl-connect` are migrated once (`atlas_legacy_migrated`). REST path is only `atlas-connect/v1`.
