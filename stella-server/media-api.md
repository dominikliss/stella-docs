# Media API — Decode Service for Foxx Cloud

**Purpose:** decode RAW / HEIC / JPEG / video posters to JPEG for **Foxx Cloud** (`cloud.foxx.family`, Laravel app on the nest, host `dedi6724`) so ImageMagick / LibRaw / RawTherapee / ffmpeg do not have to run on shared hosting. Lives on Stella under `/opt/services` like deploy-api and health-api.

**Live at:** `https://stella-media-api.foxcraft.digital` — Caddy reverse-proxies to `media-api:8080` on `edge`. No host port published. Infra: [`infrastructure.md`](infrastructure.md).

**Two ways to hand over a file:**

| Endpoint | Input | Output |
|----------|-------|--------|
| `POST /convert` | original bytes in the request body | JPEG in the response body |
| `POST /convert-remote` | Storage Box `host`, `user`, `path` in the query (empty body); Stella pulls the file | JPEG in the response body, or with `dest` stored on the Storage Box and a small JSON returned |

---

## Verification status

**Verified 2026-10-18:**
- Image builds (RawTherapee, exiftool, openssh-client included).
- `GET /health` returns `status` `ok`; tools `convert`, `identify`, `ffmpeg`, `ffprobe`, `dcraw`, `dcraw_emu`, `rawtherapee-cli`, `exiftool`, `scp`, `sftp` all present; HEIC, DNG, CR2, CR3, NEF, ARW, RAF readable by ImageMagick; `profiles.cameras` = `Sony ZV-E10`, `profiles.template` = true; `remote.enabled` = true.
- One real Sony ZV-E10 ARW through `POST /convert?op=master`: `X-Render: rawtherapee`, `X-Profile: Sony ZV-E10 Camera Standard.dcp`, 6016x4016, embedded profile `RTv4_sRGB`, about 10 s round trip.
- From inside the container: `sftp` to the Storage Box works (key, pinned `known_hosts`, port 23); directory listing returned.

**UNVERIFIED:**
- `POST /convert-remote` end to end (fetch, render, write-back with `dest`) — being tested through the Foxx Cloud app.
- Colour/detail match of the Stella output vs. the RawTherapee 5.13 Mac export. Debian's RawTherapee is older; the Stella JPEG was 6.8 MB vs 5.2 MB on the Mac.
- Styles other than `standard`; camera bodies other than ZV-E10.
- Real HEIC / DNG / CR3 files; `poster`, `preview`, `tile` with real files through the API.
- Behaviour under bulk load (`load_1m` vs Atlas thresholds). Ollama is capped to cores 0–5; media-api to 6–11, 2 workers.
- Which other Storage Boxes the copied key can reach (see Security).

**Not built:** `bake` (apply look edits; needs MediaLookBaker input format).

---

## Auth

Every `POST` is signed with a shared secret (HMAC-SHA256). `GET /health` has no auth. The secret env var is `MEDIA_HMAC_SECRET` (value not recorded here). It must exist in `/opt/services/media-api/.env` (`chmod 600`), in Foxx Cloud's `.env` on the nest, and in the password manager.

Headers:

| Header | Meaning |
|--------|---------|
| `X-Timestamp` | Unix seconds. Max skew 300 s. |
| `X-Signature` | Hex HMAC-SHA256 over the message below |

Message (newline separated, no trailing newline):

```
{timestamp}\nPOST\n{path}\n{raw query string}\n{length}
```

- `{path}` is `/convert` or `/convert-remote`.
- `{raw query string}` is the query exactly as sent, without the `?`. Sign the same string you send; do not re-encode or reorder it afterwards.
- `{length}` is the body length: the byte size of the file for `/convert`, always `0` for `/convert-remote`.
- Encode the query as RFC 3986 (`%20` for spaces). A literal `+` is decoded by the server as a space, so a `+` in a path must be sent as `%2B`.

Python reference:

```python
import hmac, hashlib, os, time

secret = os.environ["MEDIA_HMAC_SECRET"].encode()
ts = str(int(time.time()))
msg = f"{ts}\nPOST\n/convert\n{query}\n{len(body)}".encode()
sig = hmac.new(secret, msg, hashlib.sha256).hexdigest()
```

---

## Endpoints

### `GET /health`

No auth. Returns `status` (`ok` / `degraded` when a tool is missing), `workers`, `tools` (presence of each binary), `formats` (readable by ImageMagick), `profiles` (`cameras` list, `template` present) and `remote.enabled` (true when the allowlist, SSH key and `known_hosts` are present). The allowlist itself is not exposed.

### `POST /convert` — file in the body

`POST /convert?op=<op>&name=<original file name>[&style=<style>]`

| Parameter | Required | Meaning |
|-----------|----------|---------|
| `op` | yes | `preview`, `tile`, `master`, `poster` |
| `name` | yes | Original file name. Only the extension matters: it decides the decode path. |
| `style` | no | DCP style for `master` on RAW (default `standard`). Ignored by other ops. |

- Body = original bytes. `Content-Length` required (no chunked). Max 512 MiB (`MEDIA_MAX_BYTES`).
- Response 200 `image/jpeg`.

### `POST /convert-remote` — file stays on the Storage Box

`POST /convert-remote?op=<op>&host=<host>&user=<user>&path=<path>[&style=<style>][&dest=<path>][&dest_user=<user>]`

| Parameter | Required | Meaning |
|-----------|----------|---------|
| `op` | yes | as above |
| `host` | yes | Storage Box host, e.g. `u691793.your-storagebox.de` (lowercased) |
| `user` | yes | Storage Box user (lowercased) |
| `path` | yes | Absolute path of the original on the box. Must start with `/`, no `..` segment, no control characters, max 1000 chars. The extension decides the decode path. |
| `style` | no | as above |
| `dest` | no | Absolute path for the result JPEG on the box. Must end in `.jpg` or `.jpeg`, characters `A-Z a-z 0-9 space _ . , ( ) @ + = / -`, no `..`, max 900 chars. If set, the JPEG is stored there instead of being returned. |
| `dest_user` | no | User that writes `dest`. Defaults to `user`. Same `host`. |

- Body is empty. Sign with length `0`.
- Without `dest`: response 200 `image/jpeg`, same as `/convert`.
- With `dest`: response 200 `application/json`, no image:

```json
{"stored": true, "dest": "/derivatives/2026/x.jpg", "bytes": 6815570, "width": 6016, "height": 4016, "render": "rawtherapee", "profile": "Sony ZV-E10 Camera Standard.dcp"}
```

- `render` / `profile` are `null` for ops other than `master`; `profile` is `null` when `render` is `default`.
- Write-back creates missing parent folders, uploads to `<dest>.part`, removes an existing `dest`, then renames. An existing `dest` is overwritten.
- `dest` must differ from `path` when `dest_user` equals `user`, and must end in `.jpg`/`.jpeg`, so an original RAW can never be overwritten.

---

## Ops

| `op` | Output |
|------|--------|
| `preview` | Max edge 1280, q82, `-auto-orient`, sRGB, stripped |
| `tile` | 720×720 centre crop, q74, `-auto-orient`, sRGB, stripped |
| `master` | Full resolution, q92, sRGB; rejected if an edge > 12000 px (matches Foxx Cloud's `jpegBytesOk()`) |
| `poster` | One ffmpeg frame at 1 s (fallback 0 s), `-q:v 3` |

### `master` for RAW

1. `exiftool -s3 -Make -Model` reads the camera. The folder `profiles/<Make Model>` is matched case-insensitively (`SONY` + `ZV-E10` → `Sony ZV-E10`).
2. The DCP is the file in that folder whose name ends with ` <style>.dcp` (e.g. `Sony ZV-E10 Camera Standard.dcp`).
3. `main.py` copies `profiles/template.pp3` into the scratch dir, replaces the `InputProfile=` line with `file:<dcp path>` and runs:
   `rawtherapee-cli -o rt.jpg -p job.pp3 -j92 -js3 -Y -c <src>` with `HOME` set to the scratch work dir.
4. Result: full resolution (ZV-E10: 6016×4016), sRGB profile embedded (`RTv4_sRGB`), q92, 4:4:4. `X-Render: rawtherapee`, `X-Profile: <dcp file>`.

**Fallback:** if the camera has no profile folder, no matching DCP, or no `template.pp3`, the old path is used (`dcraw_emu -w -T -6 -o 1 -q 3` + ImageMagick, `-auto-orient`, sRGB, q92) and `X-Render: default`. If `rawtherapee-cli` itself fails there is no silent fallback; the request errors with `tool_failed`.

`preview` and `tile` for RAW always use the fast `dcraw_emu` path (quality 0). Only `master` uses RawTherapee.

**Styles** (depend on the DCP files present): `standard` (default), `vivid`, `portrait`, `landscape`, `monochrome`. Value rules: lowercase letters, 3–20 chars.

### Response headers (image responses)

| Header | When | Meaning |
|--------|------|---------|
| `X-Src-Width` / `X-Src-Height` | `preview`, `tile`, `master` (not `poster`) | Pixel size of the decoded source (for RawTherapee: of the rendered master) |
| `X-Render` | `master` | `rawtherapee` or `default` |
| `X-Profile` | `master` with `X-Render: rawtherapee` | DCP file name used |

---

## Errors

FastAPI wraps errors as `{"detail": ...}`. Tool failures carry `{"detail": {"error": "<code>", "detail": "<message>"}}`.

| Code | When |
|------|------|
| 400 | `/convert`: body shorter than `Content-Length` |
| 401 | Bad or missing timestamp / signature |
| 403 | `/convert-remote`: `user@host` (or `dest_user@host`) does not match the allowlist |
| 411 | `/convert`: no `Content-Length` |
| 413 | `/convert`: body empty or larger than `MEDIA_MAX_BYTES` / larger than its `Content-Length` |
| 422 | Unknown `op`; invalid file name, `style`, `host`, `user`, `path`, `dest`, `dest_user`; `dest` equals source; `too_large` (source > `MEDIA_MAX_BYTES` or an edge > 12000 px); `tool_failed` (decoder error, message in `detail`) |
| 502 | `/convert-remote`: `fetch_failed` (original not readable: auth, path, rights) or `store_failed` (write-back failed) — `detail` holds the scp/sftp message |
| 503 | Busy (all workers taken for more than 60 s; `Retry-After: 30`), or `/convert-remote` not configured |
| 504 | Timeout |

Check order on `/convert-remote`: signature (401) → op → configured (503) → host/user (422) → allowlist (403) → path → style → `dest` checks → worker slot (503) → fetch (502) → render → store (502).

---

## Limits, timeouts, concurrency

| Item | Value |
|------|-------|
| Max source size | 512 MiB (`MEDIA_MAX_BYTES`), for both endpoints |
| Max edge for `master` | 12000 px |
| ImageMagick limits | memory 2 GiB, map 4 GiB, disk 8 GiB |
| Op timeouts | preview 150 s, tile 150 s, master 280 s (RawTherapee 270 s), poster 100 s |
| Fetch / store timeout | 200 s each (`MEDIA_FETCH_TIMEOUT`) |
| Concurrency | semaphore of 2 (`MEDIA_WORKERS`); waits up to 60 s, then 503 |
| Scratch | tmpfs 4 GiB; the job dir is deleted in `finally`, leftovers wiped at startup |

A worker slot is taken **before** the fetch, so slow Storage Box transfers occupy a slot. A `/convert-remote` call with `op=master` and `dest` can take fetch + render + store, up to roughly 680 s in the worst case. Use a client timeout of at least 600 s for it.

---

## Colour profiles (Sony look)

The camera look (Creative Style) is not part of the RAW data. The ARW only embeds a ~1440×1080 preview JPEG, too small for `master`. Instead the Adobe **Camera Matching** DCPs are applied through RawTherapee.

Layout on Stella:

```
/opt/services/media-api/profiles/
├── template.pp3
└── Sony ZV-E10/
    ├── Sony ZV-E10 Camera Standard.dcp
    ├── Sony ZV-E10 Camera Vivid.dcp
    ├── Sony ZV-E10 Camera Portrait.dcp
    ├── Sony ZV-E10 Camera Landscape.dcp
    └── Sony ZV-E10 Camera Monochrome.dcp
```

`template.pp3` was exported from RawTherapee 5.13 (Mac): Adobe DCP as custom input profile, `ToneCurve=false` (with the DCP tone curve on, highlights blew out on top of RT's own curve), `ApplyLookTable`, `ApplyHueSatMap`, `ApplyBaselineExposureOffset` true, output `RTv4_sRGB`, working profile Rec2020, sharpening off, white balance Camera. Only the `InputProfile=` line is replaced per request.

**Add a camera:** copy its folder from Adobe's `CameraRaw/CameraProfiles/Camera/` directory (Camera Raw / DNG Converter) into `profiles/`. No rebuild needed. Cameras without a folder fall back to the old rendering.

**Licence:** the DCPs are Adobe's files. Keep them on Stella for own conversions, do not commit them to git, do not redistribute.

---

## Infrastructure

```
/opt/services/media-api/
├── Dockerfile
├── main.py
├── .env              # MEDIA_HMAC_SECRET only, chmod 600
├── profiles/         # template.pp3 + DCPs (see above)
└── ssh/              # id_ed25519, known_hosts (owner uid 10001, key chmod 600)
```

Relevant `media-api` service settings in `/opt/services/docker-compose.yml`:

- `build: ./media-api`, `container_name: media-api`, `env_file: ./media-api/.env`, network `edge`, `restart: unless-stopped`
- Volumes (both read-only): `./media-api/profiles:/app/profiles:ro`, `./media-api/ssh:/app/ssh:ro`
- `tmpfs: /scratch:size=4g,uid=10001,gid=10001`
- Limits: `cpus: "4"`, `memory: 8g`, `cpuset: "6-11"` (Ollama keeps `0-5`)
- Label `health.http=http://media-api:8080/health` (picked up by health-api auto-discovery — [`health-api.md`](health-api.md))

Environment variables:

| Variable | Default | Meaning |
|----------|---------|---------|
| `MEDIA_HMAC_SECRET` | — | Shared secret (required) |
| `MEDIA_WORKERS` | 2 | Concurrent jobs |
| `MEDIA_MAX_BYTES` | 512 MiB | Max source size |
| `MEDIA_QUEUE_WAIT` | 60 | Seconds to wait for a worker slot |
| `MEDIA_SCRATCH` | `/scratch` | Scratch dir |
| `MEDIA_MAX_EDGE` | 12000 | Max edge for `master` |
| `MEDIA_PROFILES` | `/app/profiles` | Profiles dir |
| `MEDIA_DEFAULT_STYLE` | `standard` | Default DCP style |
| `MEDIA_STORAGE_ALLOWED` | — | Comma-separated `user@host` patterns (fnmatch wildcards). Empty = `/convert-remote` disabled. Production value: `*@*.your-storagebox.de` |
| `MEDIA_STORAGE_PORT` | 23 | SSH port of the Storage Box |
| `MEDIA_SSH_KEY` | `/app/ssh/id_ed25519` | Private key |
| `MEDIA_SSH_KNOWN_HOSTS` | `/app/ssh/known_hosts` | Pinned host keys (`StrictHostKeyChecking=yes`) |
| `MEDIA_FETCH_TIMEOUT` | 200 | scp / sftp timeout in seconds |
| `MEDIA_DEST_PREFIXES` | — | Optional comma-separated prefixes `dest` must start with |
| `MAGICK_THREAD_LIMIT`, `OMP_NUM_THREADS` | 2 | Set in compose |

Dockerfile: `python:3.12-slim-bookworm`; apt `imagemagick libmagickcore-6.q16-6-extra libheif1 libde265-0 libraw-bin openssh-client rawtherapee libimage-exiftool-perl dcraw ffmpeg`; pip `fastapi uvicorn`; non-root user `media` (uid 10001). The Debian bookworm RawTherapee package is older than 5.13; unknown `.pp3` keys are ignored.

**Network / firewall:** port 443 is filtered port-wide in `DOCKER-USER`; only the nest (`23.88.90.12`) and one team IP (`194.126.177.181`) are allowed. Callers from anywhere else are blocked before they reach media-api. Outbound port 23 from the container to the Storage Box works (verified).

**Certificate:** Let's Encrypt production, issued 2026-10-18, expires 2027-01-16. Caddy issuer block keeps `propagation_delay 45s` and `propagation_timeout 3m`. See [`infrastructure.md`](infrastructure.md).

**Backup:** the whole `/opt/services` tree (including `media-api/`, `profiles/` and `ssh/`) is in the nightly restic run. First run that includes it: 2026-10-19 03:00 (UNVERIFIED; check `/opt/services/backup/last-run.json`). See [`backup.md`](backup.md).

---

## Security

- **HMAC** protects every `POST`; the query string (including `host`, `user`, `path`, `dest`) is part of the signature.
- **Allowlist:** `/convert-remote` only connects to hosts matching `MEDIA_STORAGE_ALLOWED` (production: any user on `*.your-storagebox.de`). Host key checking is strict against the pinned `known_hosts`.
- **Key:** the container uses a copy of Stella's default SSH key (`~/.ssh/id_ed25519`, no passphrase). This replaces the earlier design where media-api had no Storage Box access. A decoder exploit in the container could now use that key. **UNVERIFIED:** which other Storage Boxes (e.g. the restic backup box) this key also unlocks. Safer options: a dedicated key only on the media box, and Storage Box sub-accounts (read-only for originals, read-write for derivatives, passed as `user` / `dest_user`).
- **Originals are protected in code:** `dest` must end in `.jpg`/`.jpeg` and must not equal the source path. The Storage Box rights are the real protection; use a read-only account for `user` if possible.
- Path and name validation: no `..`, no control characters, no option-like values; the scratch file name is fixed (`src<ext>`).

---

## Client guide (Foxx Cloud, `cloud.foxx.family`)

Rules:

1. Call from the nest (`23.88.90.12`); other IPs are blocked.
2. Prefer `/convert-remote` when the original is on the Storage Box. Use `/convert` for files that are not there.
3. For `/convert`, send the **original** as body (not a preview), with `Content-Length`; `name` must carry the real extension (`original.arw`) so RAW files reach the RAW path.
4. Sign exactly the query string you send (RFC 3986, `%2B` for `+`).
5. For `/convert-remote` with `dest`, expect JSON, not an image. Read `render`: `rawtherapee` means the Sony profile was applied, `default` means the camera has no profile folder.
6. Client timeout: 300 s for `/convert`, at least 600 s for `/convert-remote` with `master` + `dest`.
7. Never log the secret.

Error handling:

| Status | Action |
|--------|--------|
| 503 | Retry once after `Retry-After` seconds, then fall back to the local convert |
| 502, 504, connection error | Fall back to the local convert; log `detail` |
| 401 | Check clock skew (> 300 s) and secret; no retry |
| 403, 422 | Configuration or input error; no retry (`too_large` = source > 512 MiB or edge > 12000 px) |

PHP (Laravel) sketch:

```php
<?php

namespace App\Services;

use GuzzleHttp\Psr7\Utils;
use Illuminate\Http\Client\Response;
use Illuminate\Support\Facades\Http;

class MediaApi
{
    private static function signed(string $path, string $query, int $length): array
    {
        $ts = (string) time();
        $sig = hash_hmac('sha256', "{$ts}\nPOST\n{$path}\n{$query}\n{$length}", config('services.media_api.secret'));
        return ['X-Timestamp' => $ts, 'X-Signature' => $sig];
    }

    private static function query(array $params): string
    {
        return http_build_query(array_filter($params, fn ($v) => $v !== null), '', '&', PHP_QUERY_RFC3986);
    }

    // File in the body. Returns the JPEG and metadata, or null (caller falls back to local convert).
    public static function convert(string $op, string $localPath, string $ext, string $style = 'standard'): ?array
    {
        $size = filesize($localPath);
        $query = self::query(['op' => $op, 'name' => 'original.' . strtolower($ext), 'style' => $style]);

        for ($attempt = 0; $attempt < 2; $attempt++) {
            $res = Http::withHeaders(self::signed('/convert', $query, $size))
                ->withBody(Utils::streamFor(fopen($localPath, 'rb')), 'application/octet-stream')
                ->timeout(300)
                ->post(config('services.media_api.url') . '/convert?' . $query);

            if ($res->status() === 503 && $attempt === 0) {
                sleep((int) $res->header('Retry-After') ?: 30);
                continue;
            }
            return $res->ok() ? self::image($res) : null;
        }
        return null;
    }

    // File stays on the Storage Box. With $dest the result is stored there and metadata JSON is returned.
    public static function convertRemote(
        string $op,
        string $host,
        string $user,
        string $path,
        string $style = 'standard',
        ?string $dest = null,
        ?string $destUser = null
    ): ?array {
        $query = self::query([
            'op' => $op, 'host' => $host, 'user' => $user, 'path' => $path,
            'style' => $style, 'dest' => $dest, 'dest_user' => $destUser,
        ]);

        for ($attempt = 0; $attempt < 2; $attempt++) {
            $res = Http::withHeaders(self::signed('/convert-remote', $query, 0))
                ->withBody('', 'application/octet-stream')
                ->timeout(600)
                ->post(config('services.media_api.url') . '/convert-remote?' . $query);

            if ($res->status() === 503 && $attempt === 0) {
                sleep((int) $res->header('Retry-After') ?: 30);
                continue;
            }
            if (!$res->ok()) {
                return null;
            }
            return $dest !== null ? $res->json() : self::image($res);
        }
        return null;
    }

    private static function image(Response $res): array
    {
        return [
            'jpeg' => $res->body(),
            'render' => $res->header('X-Render') ?: null,
            'profile' => $res->header('X-Profile') ?: null,
            'width' => (int) $res->header('X-Src-Width'),
            'height' => (int) $res->header('X-Src-Height'),
        ];
    }
}
```

Config: `services.media_api.url` = `https://stella-media-api.foxcraft.digital`, `services.media_api.secret` = `MEDIA_HMAC_SECRET` from `.env`.

---

## Smoke tests (on Stella)

Health:

```bash
docker exec media-api python -c "import urllib.request;print(urllib.request.urlopen('http://127.0.0.1:8080/health').read().decode())"
```

Storage Box reachable from the container:

```bash
echo "ls" | docker exec -i media-api sftp -q -P 23 -i /app/ssh/id_ed25519 -o BatchMode=yes -o StrictHostKeyChecking=yes -o UserKnownHostsFile=/app/ssh/known_hosts -b - <user>@<host>
```

`/convert-remote` with write-back:

```bash
docker exec -e RAW_PATH='/path/to/file.arw' -e DEST_PATH='/derivatives-test/file.jpg' media-api python -c '
import os,sys,time,hmac,hashlib,urllib.request,urllib.parse
q=urllib.parse.urlencode({"op":"master","host":"<host>","user":"<user>","path":os.environ["RAW_PATH"],"dest":os.environ["DEST_PATH"]},quote_via=urllib.parse.quote)
ts=str(int(time.time()))
sig=hmac.new(os.environ["MEDIA_HMAC_SECRET"].encode(),f"{ts}\nPOST\n/convert-remote\n{q}\n0".encode(),hashlib.sha256).hexdigest()
r=urllib.request.Request("http://127.0.0.1:8080/convert-remote?"+q,data=b"",method="POST",headers={"X-Timestamp":ts,"X-Signature":sig})
print(urllib.request.urlopen(r,timeout=600).read().decode())
'
```

---

## Open items

- Foxx Cloud: PHP client (MediaRaster) using `/convert-remote` with `dest`; fallback to local convert; base URL and `MEDIA_HMAC_SECRET` in the nest's `.env`. In progress.
- Verify `/convert-remote` end to end (fetch and write-back) and compare the master with the Mac export.
- Decide on a dedicated SSH key and Storage Box sub-accounts for the container (see Security).
- Implement `bake` once MediaLookBaker's input format is known.
- Test real DNG / CR3 / HEIC and other styles through the API.
- Check Atlas shows media-api (container, http, cert) after its next poll.
- Confirm the 2026-10-19 03:00 backup succeeded.
- Watch `load_1m` during bulk RAW runs (Ollama cpuset `0-5`, media-api `6-11`, 2 workers).
