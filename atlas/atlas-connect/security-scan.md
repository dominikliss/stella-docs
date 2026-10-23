# Atlas Connect — infection scan

Atlas starts and drives every scan. The WordPress site never phones home, never stores the allowlist, and never keeps a scan history. Atlas is the system of record.

Index and vault fields: [`README.md`](README.md). Login: [`login.md`](login.md). Backups: [`backups.md`](backups.md).

The plugin **collects**. Atlas **decides** `clean` / `suspicious` / `infected`.

---

## What this answers

“Has this site grown unauthorized persistence?” — a webshell, a changed core file, PHP in `uploads/`, a rogue admin, a new drop-in / mu-plugin.

It does **not** answer “is this plugin outdated?” or “is file editing on?”. No Patchstack / CVE call lives on the site.

---

## Policy

- Only Atlas can start a scan. WP admin has no Start button and no results table.
- Send the **allowlist** on **start** only (JSON body). The plugin keeps it on the running job, then drops it. Never a standing `wp_options` list.
- One running scan per site. A second start is `409` until the first is `complete` or `error`, or the job TTL expires (default 24h).
- Drive `/security/scan/chunk` **sequentially**. Do not parallelize the same `job_id`.
- HTTP client timeout: **at least 60 seconds** per chunk (the site aims for ~40s of work).
- Do **not** send yesterday’s full PHP inventory on start — that POST will blow `post_max_size`. Atlas diffs locally.
- Never log file bodies. The plugin never returns them.

---

## What Atlas must store

| Data | Why |
|---|---|
| Last **trusted** PHP inventory (`path` + `sha256` + `mtime` + `size`) | Every later scan is a diff against this |
| Allowlist (`path` + `sha256`) | Human-approved exceptions; sent on the next start |
| Last report + Atlas verdict | UI + alerts |
| Scan history | Same idea as backup history |

Encrypt nothing extra beyond the site `secret_key` you already store. Inventory and allowlist are paths and hashes.

---

## Start

```http
POST /wp-json/atlas-connect/v1/security/scan/start
Content-Type: application/json

{
    "secret": "{atlas_secret_key}",
    "allowlist": [
        { "path": "wp-content/object-cache.php", "hash": "{sha256 hex}" }
    ]
}
```

| Field | Required | Notes |
|---|---|---|
| `secret` | yes | Site secret. JSON body, not query string |
| `allowlist` | no | Array of `{ path, hash }`. `hash` is **64-char SHA-256 hex**. `filename` is accepted as an alias for `path`. Missing / invalid entries are skipped |
| `allowlist[].path` | per entry | Relative to `ABSPATH`, forward slashes, no leading `/`, no `..`. Example: `wp-content/mu-plugins/host.php` |
| `allowlist[].hash` | per entry | SHA-256 of the file **contents**. Path-only is rejected |

Allowlist is read from the **JSON/form body only**. Query-string `allowlist` is ignored.

A file is ignored in **findings** only when **both** path and hash match. If the file is replaced, it shows up again.

---

## Drive until done

```
status = POST /security/scan/start
while status.status == "running":
    status = POST /security/scan/chunk   # same secret + job_id
    append status.inventory_delta to this run's inventory
```

Stop when `status` is `complete` or `error`.

```http
POST /wp-json/atlas-connect/v1/security/scan/chunk
Content-Type: application/json

{
    "secret": "{atlas_secret_key}",
    "job_id": "{job_id}"
}
```

On `phase=hash` chunks, persist `inventory_delta` immediately. The final `report` does **not** repeat the full inventory.

---

## Status (optional UI poll)

Does **not** process a chunk.

```http
POST /wp-json/atlas-connect/v1/security/scan/status/{job_id}
Content-Type: application/json

{
    "secret": "{atlas_secret_key}"
}
```

`job_id` stays in the URL path (`[a-zA-Z0-9]+`, 12 random chars) — it is not a secret.

After `complete`, status still returns `report` until the job TTL (24h). It never returns `inventory_delta` (only hash chunks do). There is no cancel route.

**Changed 2026-10-23:** this used to be `GET …/security/scan/status/{job_id}?secret=…`. `GET` now returns `404 rest_no_route`, and a query-string secret is ignored (`401`). Build the Atlas driver against `POST` from the start.

---

## Job payload (start / chunk / status)

```json
{
    "job_id":       "AbCdEfGh1234",
    "status":       "running",
    "phase":        "hash",
    "files_total":  4200,
    "files_hashed": 1800,
    "progress":     52,
    "error":        "",
    "inventory_delta": [
        { "path": "wp-content/plugins/hello.php", "sha256": "…", "mtime": 1727791200, "size": 4164 }
    ]
}
```

On `status=complete` the same object also has:

```json
{
    "generated_at": "2026-10-22T23:00:00+00:00",
    "report": { }
}
```

| Field | Meaning |
|---|---|
| `status` | `running` \| `complete` \| `error` |
| `phase` | `inventory` \| `hash` \| `analyze` \| `done` |
| `progress` | 0–100, approximate |
| `files_total` | Candidate files (PHP-like + specials + all files under `wp-admin` / `wp-includes` / site root) |
| `files_hashed` | How many of those have been hashed |
| `inventory_delta` | Only on hash chunks. PHP-like + special files (`wp-config.php`, `.htaccess`, `.user.ini`, `web.config`). **Not** every core JS/CSS file, and not code-free PHP files (see below) |
| `error` | Set when `status` is `error` |
| `report` | Only when `status` is `complete` |

Never includes the site secret, the allowlist, or file contents.

---

## HTTP errors

| Status | Body | When |
|---|---|---|
| 401 | `{ "code": "rest_forbidden", "message": "Sorry, you are not allowed to do that.", "data": { "status": 401 } }` | Auth failed (SSL, IP, or secret — see [`README.md`](README.md#authentication-every-plugin-route)) |
| 400 | `{ "error": "Missing job_id" }` | chunk without `job_id` |
| 409 | `{ "error": "A scan is already running." }` | start while another scan runs |
| 404 | `{ "error": "Job not found" }` | Unknown or expired `job_id` |
| 404 | `{ "code": "rest_no_route", … }` | Sent as `GET` (e.g. old status call), or the `security` module / plugin is off |

Route errors are `{ "error": "…" }`. Auth and routing errors are WordPress's `{ "code", "message", "data" }`. Parse with `data.error ?? data.message`.

If WordPress.org checksums cannot be fetched, the job ends as `error` (do **not** treat the site as clean).

---

## Phases

| Phase | Work |
|---|---|
| `inventory` | Walk the tree. Collect PHP-like files everywhere, all files in `wp-admin` / `wp-includes` / site root, plus `wp-config.php` (including the parent-dir copy), `.htaccess`, `.user.ini`, `web.config` |
| `hash` | SHA-256 every candidate; MD5 as well for core-area files. Stream PHP rows as `inventory_delta` |
| `analyze` | Fetch `https://api.wordpress.org/core/checksums/1.0/?version={wp}&locale={locale}` for `en_US` plus every installed locale (site locale, `$wp_local_package`, `get_available_languages()`), cached for a week per version+locale. Build findings, collect persistence, apply allowlist, wipe scratch + allowlist |
| `done` | `status=complete` |

Skipped directories: `.git`, `.svn`, `node_modules` (exact name), plus any directory whose name **starts with** `atlas-backup-tmp`, `atlas-scan-tmp`, `.atlas-…`, or `.dlc-…` (the plugin's own scratch, including the random-suffix fallback dirs in uploads). Symlinks are not followed. Files over 32 MB are not hashed.

---

## Report (complete only)

```json
{
    "generated_at": "2026-10-22T23:00:00+00:00",
    "core": {
        "version": "6.8.3",
        "locale": "de_DE",
        "locales": ["en_US", "de_DE"],
        "checked": 1850,
        "mismatched": [
            { "path": "wp-includes/functions.php", "sha256": "…", "mtime": 1727791200, "size": 12000 }
        ],
        "unexpected": [
            { "path": "wp-admin/about.php.bak", "sha256": "…", "mtime": 1727791200, "size": 80 }
        ],
        "missing": ["wp-admin/css/colors/blue/colors.min.css"]
    },
    "php_in_uploads": [
        { "path": "wp-content/uploads/2024/03/x.php", "sha256": "…", "mtime": 1727791200, "size": 512 }
    ],
    "special_files": {
        "wp-config.php": { "sha256": "…", "mtime": 1727791200, "size": 3200 },
        ".htaccess":     { "sha256": "…", "mtime": 1727791200, "size": 400 },
        ".user.ini":     null,
        "web.config":    null
    },
    "admins": [
        { "id": 1, "login": "dominik", "registered": "2024-01-02 10:00:00" }
    ],
    "dropins": ["object-cache.php"],
    "mu_plugins": ["host-mu.php"],
    "cron": [
        { "hook": "wp_version_check", "schedule": "twicedaily" }
    ],
    "urls": {
        "home": "https://client.com/",
        "siteurl": "https://client.com"
    }
}
```

| Block | Meaning |
|---|---|
| `core.locales` | Locales whose checksums were merged. `en_US` is required (fetch failure = job `error`); other locales without an official package are skipped |
| `core.mismatched` | File is in the official checksum list; MD5 matches **none** of the fetched locales. Allowlisted path+SHA-256 omitted. Nothing under `wp-content/` is checked against core checksums: bundled themes (Twenty Twenty-*), bundled plugins (Akismet, Hello Dolly) and translations are updated independently of core. Their PHP files are still in `inventory_delta`, so the baseline diff covers them |
| `core.unexpected` | Extra file in `wp-admin`, `wp-includes`, or the site root that is not in any locale's checksum list. `wp-config.php` / `.htaccess` / `.user.ini` / `web.config` are excluded (they live under `special_files`) |
| `core.missing` | Core file shipped by **every** fetched locale but not on disk (broken install more often than malware) |
| `php_in_uploads` | PHP-like file under uploads (`.php`, `.phtml`, `.php5`, `.phar`, …). Allowlisted and code-free files omitted |
| `special_files` | SHA-256 of those four names, or `null` if absent |
| `admins` | Administrators: `id`, `login`, `registered`. No emails |
| `dropins` | Present drop-in filenames in `wp-content/` |
| `mu_plugins` | Must-use plugin basenames |
| `cron` | Unique WP-Cron hook names + schedule |
| `urls` | `home_url` and `siteurl` option |

Finding objects are `{ path, sha256, mtime, size }` only. No snippets, no file bodies.

**Code-free PHP files are never reported.** A PHP-like file is code-free when it is empty, or contains only `<?php` / `?>` tags, comments, and whitespace (e.g. `<?php // Silence is golden`). Such files are left out of `core.mismatched`, `core.unexpected`, `php_in_uploads`, and `inventory_delta`. Anything else counts as code, including `<?=`, short `<?` tags, a lone `;`, and HTML after a `?>`. Files over 64 KB are always treated as code. If code is later added to one of these files, it shows up as a new PHP path and is flagged as usual.

**Google Search Console verification files** (`google<hex>.html`) are not reported in `core.unexpected` when their content is exactly `google-site-verification: <same file name>` (surrounding whitespace allowed). Any other content, a different name in the body, or a different extension is reported as usual.

**Data files.** Paths listed in `config.php` → `security.data_files` may also start with `<?php __halt_compiler();`. PHP stops at that call, so whatever follows is data and never runs. Such a file is treated as code-free (no size limit) only while nothing but tags, comments, and whitespace come before `__halt_compiler();`. The path alone is never trusted. Today's entry: `wp-content/uploads/aios/firewall-rules/settings.php` (All In One Security firewall settings).

**Ignored changes.** Paths listed in `config.php` → `security.ignore_changes` are real PHP that plugins regenerate, so they are left out of `inventory_delta` and their hash changes never alert. Drop-ins among them are still listed under `dropins`, so a newly appearing one is still visible. Today's entry: `wp-content/advanced-cache.php` (page cache drop-in). Malicious code inside it is **not** detected by the baseline diff.

---

## How Atlas should score the site

Do this in Atlas. The plugin does not return a verdict.

### First scan (no baseline yet)

| Signal | Verdict |
|---|---|
| `php_in_uploads` non-empty (after allowlist) | **infected** |
| `core.unexpected` PHP (or any unexpected file you do not recognize) | **infected** |
| `core.mismatched` | **infected** (or allowlist a known host patch) |
| New-looking drop-in / mu-plugin you did not install | **suspicious** |
| Otherwise | **clean** (no evidence — not a guarantee) |

Store this run’s `inventory_delta` (all hash chunks concatenated) as the trusted baseline **after** a human has reviewed an `infected` / `suspicious` first run, or automatically if the first run is `clean`.

### Every later scan

Diff against the last trusted inventory **and** the last persistence snapshot:

| Change Atlas did not make | Verdict |
|---|---|
| New PHP path, or SHA-256 change on a path that used to match official core / yesterday | **infected** |
| New administrator | **infected** |
| `special_files` hash change for `wp-config.php` / `.htaccess` / `.user.ini` | **infected** |
| `urls.siteurl` / `home` changed | **infected** |
| New drop-in or mu-plugin | **suspicious** |
| New cron hook | **suspicious** |
| Inventory + persistence unchanged, core still matches | **clean** |

Alert only on **new** high-confidence findings, not on the same allowlisted file every day.

---

## Allowlist UX

When an operator marks a finding “this is ours”:

1. Append `{ path, hash: sha256 }` to that site’s allowlist.
2. Do **not** allowlist by path alone.
3. Next day’s `start` sends the full allowlist (it stays small: exceptions, not the inventory).
4. If the file changes, the hash no longer matches and it becomes a finding again.

---

## Suggested Atlas driver (TypeScript)

```ts
type AllowItem = { path: string; hash: string };

type InventoryRow = { path: string; sha256: string; mtime: number; size: number };

type ScanJob = {
  job_id: string;
  status: 'running' | 'complete' | 'error';
  phase: string;
  progress: number;
  files_total: number;
  files_hashed: number;
  error: string;
  inventory_delta?: InventoryRow[];
  generated_at?: string;
  report?: Record<string, unknown>;
};

async function atlasPost(siteUrl: string, path: string, body: object): Promise<ScanJob> {
  const res = await fetch(`${siteUrl.replace(/\/$/, '')}/wp-json/atlas-connect/v1${path}`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(body),
    signal: AbortSignal.timeout(90_000),
  });
  const data = await res.json().catch(() => ({}));
  // Route errors use `error`; auth (401) and routing (404 rest_no_route) errors use WP's `message`.
  if (!res.ok) throw new Error(data.error ?? data.message ?? `HTTP ${res.status}`);
  return data as ScanJob;
}

export async function scanStatus(siteUrl: string, secret: string, jobId: string): Promise<ScanJob> {
  // POST, secret in the body — GET with ?secret= is gone.
  return atlasPost(siteUrl, `/security/scan/status/${encodeURIComponent(jobId)}`, { secret });
}

export async function runScan(
  siteUrl: string,
  secret: string,
  allowlist: AllowItem[] = [],
): Promise<{ job: ScanJob; inventory: InventoryRow[] }> {
  let job = await atlasPost(siteUrl, '/security/scan/start', { secret, allowlist });
  const inventory: InventoryRow[] = [];

  while (job.status === 'running') {
    job = await atlasPost(siteUrl, '/security/scan/chunk', {
      secret,
      job_id: job.job_id,
    });
    if (job.inventory_delta?.length) {
      inventory.push(...job.inventory_delta);
    }
  }

  return { job, inventory };
}
```

On `complete`, persist `inventory`, persist `job.report`, compute the verdict, alert if it is new.

Suggested Atlas surface (not built yet):

| Method | Path | Purpose |
|---|---|---|
| `POST` | `/api/wp-sites/{id}/scans` | Start + queue drive (or run sync like backups) |
| `GET` | `/api/wp-sites/{id}/scans` | History |
| `GET` | `/api/wp-sites/{id}/scans/{scan}` | One report + verdict |
| `POST` | `/api/wp-sites/{id}/scans/{scan}/allowlist` | Add `{ path, hash }` from a finding |

Daily: same pattern as `wp-sites:drive-backups` — one scheduled command that starts a scan per site and chunks until `complete`.

---

## Plugin internals (not required to implement Atlas)

`config.php` → `modules.security` is **on**. Routes register only when that flag is true.

Auth is `atlas_secret_authentication()`: SSL → client IP in `ip_whitelist` (`2.31.42.13`, `95.217.144.93`, `172.20.0.1` for Stella-internal hairpin; `X-Forwarded-For` trusted only from `trusted_proxies`) → `secret` from the JSON body, compared as SHA-256. Same `401` as login/backup. Details: [`README.md`](README.md#authentication-every-plugin-route).

Job state: option `atlas_scan_job_{job_id}` (not autoloaded). Active id: `atlas_scan_active_job`. TTL default 86400s. Scratch (file list, inventory queue, hashes) in `atlas_scratch_dir( 'atlas-scan-tmp' )` — a private `0700` dir under `sys_get_temp_dir()`, or `uploads/atlas-scan-tmp-{random}/` as a fallback (see [`backups.md`](backups.md#plugin-internals-not-required-to-implement-atlas)). The old fixed `uploads/atlas-scan-tmp/` is deleted on upgrade. Daily `atlas_cleanup_cron` drops expired jobs and scratch.

Core compare uses WordPress.org **MD5** checksums. Inventory / allowlist / baseline use **SHA-256**.

v1 does **not** checksum WP.org plugins/themes. Premium / custom plugins are inventory rows only; first-scan they are not “infected”. Atlas catches them on the next diff if they change.

No local `{prefix}atlas_scan_log` table. WP admin has no Security tab; scans are started and viewed from Atlas only.
