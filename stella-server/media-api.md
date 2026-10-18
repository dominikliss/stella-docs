# Media API — Stateless Decode Service for Foxx Cloud

**Purpose:** decode RAW / HEIC / JPEG / video posters to JPEG for **Foxx Cloud** (Laravel app on the nest, host `dedi6724`) so ImageMagick / LibRaw / ffmpeg do not have to run on shared hosting. Lives on Stella under `/opt/services` like deploy-api and health-api.

**Live at:** `https://stella-media-api.foxcraft.digital` — Caddy reverse-proxies to `media-api:8080` on `edge`. No host port published. Caller is the nest (`foxxqn@dedi6724`, outgoing IP `23.88.90.12`, already ACCEPTed on port 443). Infra: [`infrastructure.md`](infrastructure.md).

---

## Verification status

**Verified 2026-10-18:**
- Image builds.
- `GET /health` returns `status` `ok` with all tools present (`convert`, `identify`, `ffmpeg`, `ffprobe`, `dcraw`) and HEIC, DNG, CR2, CR3, NEF, ARW, RAF reported readable by ImageMagick.
- Signed `POST /convert?op=preview` from inside the container returned 200 with `X-Src-Width` / `X-Src-Height`.
- HTTPS `GET /health` works from the nest (`foxxqn@dedi6724`).

**UNVERIFIED:**
- Decoding of a real RAW (DNG / CR3 / …) or HEIC file through the API.
- `master`, `poster`, and `tile` ops through the API.
- Behaviour under bulk load (`load_1m` vs Atlas thresholds). Ollama is capped to cores 0–5; media-api to 6–11, 2 workers.

**Not built:**
- `bake` (apply look edits; needs MediaLookBaker input format).
- The PHP client in Foxx Cloud (signing, upload, fallback to local convert) — in MediaRaster.

---

## Why this shape

**Stateless request/response, not Storage Box jobs.** The nest uploads the original in the request body and gets the JPEG back. media-api has no Storage Box access, no SSH key, and no `known_hosts` for this service. A decoder exploit therefore cannot reach any Storage Box (including the restic backup box).

Rejected for now: async jobs where media-api pulls originals from the Storage Box via SFTP and writes derivatives back. Revisit only if upload size/time from the nest or HTTP timeouts on `master` become a problem.

**Own container, not an endpoint on stella-api.** stella-api is chat-only and runs with `network_mode: host`. Media jobs need their own CPU/memory caps and isolation from untrusted file decoding. See [`stella-api.md`](stella-api.md).

**HMAC shared secret, not the SSH-signature scheme of deploy-api / health-api.** Shared hosting often lacks `ssh-keygen -Y`. Secret env var is `MEDIA_HMAC_SECRET` (value not recorded here). Must exist in `/opt/services/media-api/.env` (`chmod 600`), Foxx Cloud’s `.env` on the nest, and the password manager.

---

## Architecture

```
/opt/services/media-api/
├── Dockerfile
├── main.py
└── .env          # MEDIA_HMAC_SECRET only, chmod 600
```

```yaml
# /opt/services/docker-compose.yml (relevant service, above top-level networks:)
media-api:
  build: ./media-api
  container_name: media-api
  env_file:
    - ./media-api/.env
  environment:
    - MEDIA_WORKERS=2
    - MAGICK_THREAD_LIMIT=2
    - OMP_NUM_THREADS=2
  tmpfs:
    - /scratch:size=4g,uid=10001,gid=10001
  deploy:
    resources:
      limits:
        cpus: "4"
        memory: 8g
  cpuset: "6-11"
  labels:
    - health.http=http://media-api:8080/health
  networks:
    - edge
  restart: unless-stopped
```

No published ports, no volumes. Scratch is tmpfs (4 GiB, uid/gid 10001). Ollama keeps cpuset `0-5`; media-api uses `6-11`. The `health.http` label is picked up by health-api auto-discovery — [`health-api.md`](health-api.md).

### Dockerfile

`python:3.12-slim-bookworm`. Apt: `imagemagick` `libmagickcore-6.q16-6-extra` `libheif1` `libde265-0` `libraw-bin` `dcraw` `ffmpeg`. Pip: `fastapi` `uvicorn`. Non-root user `media` (uid 10001). `/scratch` dir. `ENV MAGICK_TMPDIR=/scratch TMPDIR=/scratch`. HEALTHCHECK against `127.0.0.1:8080/health`. CMD: uvicorn `main:app` on `0.0.0.0:8080`.

Debian bookworm package names (hit during the image build): `libraw23` does not exist (omit it; `libraw-bin` pulls the library); `libheif-plugin-libde265` is replaced by `libde265-0`. See [`infrastructure.md`](infrastructure.md).

---

## Auth

Headers on `POST /convert`:

| Header | Meaning |
|--------|---------|
| `X-Timestamp` | Unix seconds. Max skew 300 s. |
| `X-Signature` | Hex HMAC-SHA256 of `MEDIA_HMAC_SECRET` over `"{timestamp}\nPOST\n/convert\n{raw query string}\n{content-length}"` |

`GET /health` has no auth.

Caller-side sketch (secret from env, never logged):

```python
import hmac, hashlib, os, time

secret = os.environ["MEDIA_HMAC_SECRET"].encode()
timestamp = str(int(time.time()))
query = "op=preview&name=example.cr3"   # raw query string as sent
content_length = str(len(body))
msg = f"{timestamp}\nPOST\n/convert\n{query}\n{content_length}".encode()
signature = hmac.new(secret, msg, hashlib.sha256).hexdigest()
```

Then `X-Timestamp: {timestamp}` and `X-Signature: {signature}`.

---

## Endpoints

| Method | Path | Auth | Purpose |
|--------|------|------|---------|
| GET | `/health` | none | Liveness + tool/format inventory |
| POST | `/convert?op=preview\|tile\|master\|poster&name=<original file name>` | HMAC | Decode original bytes → JPEG |

### `GET /health`

Returns `status` (`ok` / `degraded`), `workers`, `tools` (whether `convert`, `identify`, `ffmpeg`, `ffprobe`, `dcraw` are present), and `formats` (whether HEIC, DNG, CR2, CR3, NEF, ARW, RAF are readable by ImageMagick).

### `POST /convert`

- Body = raw original bytes. `Content-Length` required (no chunked). Max 512 MiB (`MEDIA_MAX_BYTES`).
- The extension of `name` decides decoding.
- Response 200 `image/jpeg`. `X-Src-Width` / `X-Src-Height` for `preview` / `tile` / `master` (not `poster`).

| `op` | Output |
|------|--------|
| `preview` | Max edge 1280, q82 |
| `tile` | 720×720 center crop, q74 |
| `master` | Full resolution, `-auto-orient`, sRGB, q92; rejected if an edge > 12000 px (matches Foxx Cloud’s `jpegBytesOk()`) |
| `poster` | One ffmpeg frame at 1 s (fallback 0 s), `-q:v 3` |

ImageMagick limits: memory 2 GiB, map 4 GiB, disk 8 GiB.

Timeouts: preview 150 s, tile 150 s, master 280 s, poster 100 s.

Concurrency: semaphore of 2 (`MEDIA_WORKERS`). Waits up to 60 s, then 503 with `Retry-After: 30`.

The request body is streamed into the tmpfs scratch dir, converted, and the scratch dir is deleted in `finally`. Leftovers are wiped at startup.

### Errors

| Code | When |
|------|------|
| 401 | Bad signature or timestamp |
| 411 | No `Content-Length` |
| 413 | Body larger than `MEDIA_MAX_BYTES` |
| 422 | Unknown `op` / invalid file name / `too_large` / `tool_failed` |
| 503 | Busy (semaphore wait exceeded) |
| 504 | Timeout |

---

## Public route

Subdomain `stella-media-api.foxcraft.digital` — same `stella-` prefix as `stella-health-api` / `stella-deployment-api` (chose that over plain `media-api`).

DNS A record via the Hetzner Cloud API, zone `567656`, rrset name `stella-media-api`, TTL 300, value `95.217.144.93`. A first record `media-api` was created and deleted again; no certificate was ever issued for it.

Certificate: Let's Encrypt production (issuer YE1), issued 2026-10-18, expires 2027-01-16. Caddy issuer block includes `propagation_delay 45s` and `propagation_timeout 3m` — keep both; they are needed for renewals too. See [`infrastructure.md`](infrastructure.md) Issue 2 / Issue 3.

**Firewall:** no change. The nest’s outgoing IP `23.88.90.12` is already one of the two static team IPs ACCEPTed on port 443 in `DOCKER-USER` (the other is `194.126.177.181`). Tested from `foxxqn@dedi6724`: connection works. Port 443 filtering is port-wide, so no per-site `remote_ip` block was added for media-api. HMAC is the second protection layer.

---

## Backup

No extra backup config. The whole `/opt/services` tree (including `media-api/` and its `.env`) is covered by the nightly restic run. media-api holds no data; scratch is tmpfs. See [`backup.md`](backup.md).

**UNVERIFIED:** first restic run that includes media-api is 03:00 on 2026-10-19 — check `/opt/services/backup/last-run.json` for `status` success.

Caddy certificates live in the `caddy-data` volume (this includes the `stella-media-api` cert) and are **not** part of the `/opt/services` tree backup.

---

## Open items

- Foxx Cloud: PHP client in MediaRaster (signed `POST /convert`, ops preview / tile / master / poster, fallback to local convert, base URL + `MEDIA_HMAC_SECRET` in the nest’s `.env`).
- Implement `bake` once MediaLookBaker’s input format is known.
- Test a real RAW (DNG / CR3) and HEIC file through the API.
- Check Atlas shows media-api (container, http, cert) after its next poll.
- Confirm the 2026-10-19 03:00 backup succeeded.
- Watch `load_1m` during bulk RAW runs.
