# System overview — how the pieces connect

This is the **cross-system map** for foxcraft.digital: who owns what, which host talks to which, and which trust model sits on each path.

**Related docs**

- Atlas platform: [`../atlas/README.md`](../atlas/README.md)
- Atlas production host: [`../atlas/server.md`](../atlas/server.md)
- Atlas host API: [`../atlas/host-api.md`](../atlas/host-api.md)
- Atlas dev instance: [`../atlas/dev-instance.md`](../atlas/dev-instance.md)
- Atlas architecture: [`../atlas/architecture.md`](../atlas/architecture.md)
- Atlas ↔ ddashboard login: [`../atlas/auth.md`](../atlas/auth.md)
- Atlas ↔ WordPress sites: [`../atlas/atlas-connect/`](../atlas/atlas-connect/)
- ddashboard ↔ Stella (AI chat only): [`ddashboard-and-stella-server.md`](ddashboard-and-stella-server.md)
- Stella host: [`../stella-server/README.md`](../stella-server/README.md)
- Stella media-api: [`../stella-server/media-api.md`](../stella-server/media-api.md)
- Edison (coding-agent host): [`../edison/README.md`](../edison/README.md)

---

## 1. The four named systems

Servers use One Piece Vegapunk-satellite names. Firewalls, IPs, and networks use **functional** names.

| Name | What it is | Host | Owns |
|------|------------|------|------|
| **ddashboard** | Internal business app (CRM, accounting, mail, PM, AI chat) | Hetzner managed WordPress (`foxcrr`) | Relational business data, WP sessions, `dls/v1` auth tokens |
| **Atlas** | Internal ops platform (deploys, Stella health, WP uptime, IMAP migration, TrackingTime import, Trello push) | Hetzner Cloud CX33 `atlas` (`2.31.42.13`) — `https://atlas.foxcraft.digital` | Deploy history, health history, WP-site registry, IMAP job history, imported PM, Trello card ids |
| **Stella** | Dedicated AI / services machine | Hetzner dedicated | Ollama, stella-api, deploy-api, health-api, media-api, imap-sync, Caddy, .NET dev apps |
| **Edison** | Coding-agent sandbox | Hetzner Cloud (same project as Atlas) | Cursor CLI container, sshfs mounts. **Must not** run on Stella. Atlas reads Cloud API status/backups; no agent control. |

Neither Atlas nor ddashboard replaces the other:

- **ddashboard** is the product the business runs on (clients, invoices, mail, projects).
- **Atlas** is the ops console that used to live as **Werkzeuge** inside ddashboard (deploy, WP sites, mailbox copy). Those screens were removed from the theme on 2026-09-02.

Stella is a **service host**. It does not have a general-purpose UI. Atlas and ddashboard are the two HTTP clients that matter for ops and chat. **Foxx Cloud** (Laravel on the nest, host `dedi6724`) calls **media-api** for image/video decode.

---

## 2. Trust boundaries (who may call whom)

```
 Operator browser
        │
        │  HTTPS (IP-allowlisted on Atlas)
        ▼
     Atlas SPA  ────────── Bearer token ──────────►  Atlas Laravel API
        │                                                 │
        │                                                 │ 1. login / verify / refresh
        │                                                 ▼
        │                                           ddashboard
        │                                           /dls/v1/auth/*
        │
        │                                                 │ 2. signed SSH (namespace `deploy`)
        │                                                 ▼
        │                                           Stella
        │                                      deploy-api  /  health-api
        │
        │                                                 │ 3. plain HTTPS (trusted network / API keys)
        │                                                 ▼
        │                                           Stella imap-sync
        │                                           WP sites (HEAD + Atlas Connect)
        │                                           GitHub (SSH clone)
        │                                           destination SSH servers (rsync)
        │                                           TrackingTime Pro
        │                                           Trello REST v1 (key + token)
        │
        └── browser never talks to Stella or to client WP sites directly
            (except opening an Atlas Connect one-time login URL).
            The site never calls Atlas. Backups are Atlas → site → S3.


 Operator browser
        │
        │  HTTPS (WordPress session)
        ▼
    ddashboard SPA  ─── WP REST ───►  WordPress
                                          │
                                          │ server-to-server
                                          ▼
                                      Stella stella-api
                                      POST /chat/stream
                                      (+ Ollama for some agents)
```

**Hard rules**

1. The **browser never calls Stella**. Atlas, ddashboard, and Foxx Cloud proxy every Stella call server-side.
2. Atlas **does not** issue its own login passwords. Identity lives in ddashboard (`/dls/v1/auth/*`).
3. Stella **deploy-api** and **health-api** do not use bearer tokens. They verify an OpenSSH signature from Atlas’s SSH private key, principal `ddashboard` (same keypair as the old managed-host Atlas; the managed host still holds a copy).
4. Stella **media-api** uses HMAC-SHA256 (`MEDIA_HMAC_SECRET`), not SSH signatures — shared hosting on the nest often lacks `ssh-keygen -Y`. See [`../stella-server/media-api.md`](../stella-server/media-api.md).
5. Stella **stella-api** (`/chat/*`) and **imap-sync** have no HTTP auth — ingress is restricted at the network / Caddy layer.
6. Atlas itself is IP-allowlisted (`ATLAS_ALLOWED_IPS` = `194.126.177.181`, `23.88.90.12` managed host, `95.217.144.93` Stella) except `GET /api/health`. **TODO (Dominik):** remove `23.88.90.12` unless something on the managed host really calls Atlas; reason for each entry unverified.
7. Stella `DOCKER-USER` ACCEPTs Atlas (`2.31.42.13`) on 443 so signed deploy-api / health-api calls can reach Caddy. The nest (`23.88.90.12`) is already ACCEPTed on 443 (team IP) and is the media-api caller. Atlas inbound Hetzner firewall allows 22/80/443/ICMP from `194.126.177.181` only.

---

## 3. What each HTTP path is for

### Atlas → ddashboard (identity)

| Atlas call | ddashboard route | Purpose |
|------------|------------------|---------|
| `POST /api/auth/login` | `POST /dls/v1/auth/login` | Exchange username/password for access + refresh tokens |
| `POST /api/auth/refresh` | `POST /dls/v1/auth/refresh` | Rotate access token |
| `POST /api/auth/logout` | `POST /dls/v1/auth/logout` | Revoke refresh token |
| Token middleware (cache miss) | `GET /dls/v1/auth/verify` | Validate bearer token, read `{ id, email, name, roles[] }` |
| Tools → TrackingTime Import | `GET /dls/v1/atlas/clients`, `/atlas/clients/{id}`, `/atlas/users` | Slim client stub + team users. Bearer + IP allowlist (Atlas `2.31.42.13`, Stella `95.217.144.93` only). Atlas calls these; **not implemented on ddashboard yet.** [`../stella-dashboard/atlas-client-lookup.md`](../stella-dashboard/atlas-client-lookup.md) |

Details: [`../atlas/auth.md`](../atlas/auth.md).

### Atlas → Stella (ops)

| Atlas feature | Stella service | Auth | Contract |
|---------------|----------------|------|----------|
| Deploy type `stella_deploy_api` | **deploy-api** `https://stella-deployment-api.foxcraft.digital` | SSH signature, payload `deploy-{app}:{ts}` / `list-apps:{ts}` | [`../stella-server/deploy-api.md`](../stella-server/deploy-api.md), [`../atlas/deploy-pipeline.md`](../atlas/deploy-pipeline.md) |
| Monitoring cron + dashboard | **health-api** `https://stella-health-api.foxcraft.digital` | SSH signature, payload `system-health:{ts}` | [`../stella-server/health-api.md`](../stella-server/health-api.md), [`../atlas/monitoring.md`](../atlas/monitoring.md) |
| Tools → IMAP Migration | **imap-sync** `https://stella.foxcraft.digital/imap-sync` | Network restriction only | [`../stella-server/imap-sync-service.md`](../stella-server/imap-sync-service.md), [`../atlas/tools.md`](../atlas/tools.md) |
| Health-api **fallback** probes | Public Caddy hosts (`/chat/health`, `/ollama`, apps) | None | [`../atlas/monitoring.md`](../atlas/monitoring.md) |

Atlas **does not** call `stella-api` `/chat/*`. That path is ddashboard’s.

### Atlas → Atlas host API

| Caller | URL | Auth | Contract |
|--------|-----|------|----------|
| Atlas dev/prod (health), Atlas dev only (deploy) | `https://api.atlas.foxcraft.digital` | SSH signature, `system-health:{ts}` / `deploy-atlas:{ref}:{ts}` / `deploy-status:{ts}` | [`../atlas/host-api.md`](../atlas/host-api.md) |

Laravel `CheckAtlasHealth` stores `atlas_health_checks` / `atlas_service_uptime_logs` and renders the dashboard card + `/monitoring/atlas`. On atlas.dev only, `ATLAS_SELF_DEPLOY_ENABLED` exposes `POST /api/monitoring/atlas/deploy` and `GET /api/monitoring/atlas/deploy/status` (404 when off). See [`../atlas/monitoring.md`](../atlas/monitoring.md).

### ddashboard → Stella (AI)

| Feature | Stella service | Auth |
|---------|----------------|------|
| `general` agent streaming chat | **stella-api** `POST /chat/stream` | Network restriction |
| Other agents / mail analyses | Ollama (or Anthropic / OpenAI) | Network restriction / provider key |

Details: [`ddashboard-and-stella-server.md`](ddashboard-and-stella-server.md).

### Foxx Cloud (nest) → Stella (media)

| Feature | Stella service | Auth | Contract |
|---------|----------------|------|----------|
| Decode original → JPEG (`preview` / `tile` / `master` / `poster`) | **media-api** `https://stella-media-api.foxcraft.digital` | HMAC-SHA256 (`X-Timestamp` + `X-Signature`) | [`../stella-server/media-api.md`](../stella-server/media-api.md) |

PHP client in MediaRaster: in progress (prefer `/convert-remote` with `dest` when the original is on the Storage Box). `bake` is **not built**. One ZV-E10 ARW `master` via `/convert` is verified; `/convert-remote` end to end is **UNVERIFIED** (2026-10-18).

### Atlas → WordPress sites (client / own sites)

| Feature | Target | Auth |
|---------|--------|------|
| Uptime every 5 min | `HEAD https://{domain}` | None (TLS verify off) |
| One-click WP admin login | `POST https://{domain}/wp-json/atlas-connect/v1/generate-token` | Shared **Atlas Connect** secret (JSON body; site stores only its SHA-256) + Atlas IP (`2.31.42.13` / `95.217.144.93`) in the plugin's `ip_whitelist`. Administrators only. The returned link only opens from the team office IP `194.126.177.181` |
| Encrypted backup (plugin ready; Atlas driver TODO) | `POST …/atlas-connect/v1/backup/{start,chunk,cancel,status/{job_id}}` | Same secret + IP allowlist. Atlas sends S3 credentials in the JSON body; the site does not store them. Destination pinned to bucket `atlas-wp-sites-backups` on `https://{fsn1,nbg1,hel1}.your-objectstorage.com` |
| Infection scan (plugin ready; Atlas driver TODO) | `POST …/atlas-connect/v1/security/scan/{start,chunk,status/{job_id}}` | Same secret + IP allowlist. Atlas sends `{ path, hash }` allowlist on start; stores inventory + report; site does not keep results |

All plugin routes are `POST`; the former `GET …/status/{job_id}?secret=` routes are gone (2026-10-23).

Details: [`../atlas/atlas-connect/`](../atlas/atlas-connect/).

### Atlas → GitHub + SSH servers (classic deploys)

| Feature | Target | Auth |
|---------|--------|------|
| `ssh_rsync` clone | `git@github.com:{owner}/{repo}.git` | Same Atlas SSH key as a GitHub deploy key |
| `ssh_rsync` sync | `user@host:path` | Same key in `authorized_keys` |
| Branch picker / SSH folder browser | GitHub via SSH; remote `ls` via SSH | Same key |

Details: [`../atlas/deploy-pipeline.md`](../atlas/deploy-pipeline.md).

### Atlas → TrackingTime / Trello

| Feature | Target | Auth | Contract |
|---------|--------|------|----------|
| Tools → TrackingTime Import / hourly open-task sync | TrackingTime Pro API v4 | App password in Settings | [`../atlas/tools.md`](../atlas/tools.md) |
| Projects → Trello push | Trello REST v1 | API key + token in Settings | [`../atlas/projects.md`](../atlas/projects.md) |
| Notification rules → Slack | Incoming webhooks (`https://hooks.slack.com/…`) | Encrypted webhook URL per destination | [`../atlas/notifications.md`](../atlas/notifications.md) |

One-way only. Trello never writes back. Hourly Trello is a side effect of the TrackingTime open-task sync, not its own cron. Slack is outbound webhooks only.

---

## 4. Shared SSH key (the one secret that spans three jobs)

Atlas uses **one SSH private key** (Settings → GitHub, `app_settings.deploy_ssh_key_path`, fallback `DEPLOY_SSH_KEY_PATH`):

1. **GitHub** — clone private repos (`ssh_rsync`)
2. **Destination servers** — rsync
3. **Stella** — `ssh-keygen -Y sign -n deploy` for deploy-api and health-api

Stella’s `deploy-api/allowed_signers` lists this public key as principal **`ddashboard`**. health-api has its **own** file (`/opt/services/health-api/allowed_signers`) with two `ddashboard` lines (`atlas-prod` and `atlas-dev`). The same `atlas-prod` and `atlas-dev` keys are also in `/opt/atlas-api/allowed_signers` (principal `ddashboard`, namespace `deploy`). `allowed_signers_deploy` contains only the atlas-dev key. The principal name is historical; health-api hardcodes `ddashboard` / namespace `deploy`. Production Atlas reuses the old managed-host key (`/home/atlas/.ssh/id_ed25519`, comment `atlas-prod`) so the 28 deploy servers did not need new `authorized_keys`. The managed host still holds the same key and can still deploy/sign. Rotation was considered and deferred. Dev instance: [`../atlas/dev-instance.md`](../atlas/dev-instance.md).

The private key never crosses the network. Only a short-lived signed payload (`action:unix_timestamp`, 60 s replay window on Stella) is posted.

---

## 5. Data ownership

| Data | Source of truth | Replica / history |
|------|-----------------|-------------------|
| Users, roles, passwords | **ddashboard** (WordPress) | Atlas `users` + Spatie roles (synced on login / verify) |
| Clients, invoices, mail | **ddashboard** MySQL | Atlas stores a client stub (`ddashboard_id`, `tracking_time_pro_id`, `name`) after import — [`../stella-dashboard/atlas-client-lookup.md`](../stella-dashboard/atlas-client-lookup.md) |
| PM board (`dls_pm_*`) | **ddashboard** MySQL | Existing in-theme board. New TrackingTime imports go to Atlas MySQL (tasks, lists, attachments, time entries). Optional one-way Trello push from Atlas: [`../atlas/projects.md`](../atlas/projects.md) |
| Deploy configs, runs, logs | **Atlas** MySQL | Stella deploy-api keeps its own in-memory/job logs during a run |
| Stella health snapshots | **Atlas** (`stella_health_checks`, `stella_service_uptime_logs`) | health-api is live-only except last-seen container names in `seen.json`; Atlas stores the snapshots |
| WP site registry + uptime | **Atlas** (`wp_sites`, `wp_site_uptime_logs`) | Site itself does not push status (Atlas pulls) |
| IMAP migration job history | **Atlas** (`imap_migration_jobs`, **no passwords**) | Stella `imap-sync` holds live job + log in memory (lost on restart, 4 h TTL) |
| LLM inference | **Stella** Ollama | ddashboard stores chat transcripts |

---

## 6. What moved out of ddashboard (2026-09-02)

These Werkzeuge screens were deleted from the theme. Atlas is the replacement:

| Old ddashboard UI | Atlas module | Stella backend (if any) |
|-------------------|--------------|-------------------------|
| Deployment | **Deployments** | deploy-api for .NET / Docker apps; SSH rsync for PHP/WP |
| WordPress-Seiten | **WP Sites** | None (direct HEAD + Atlas Connect plugin) |
| E-Mail-Migration | **Tools → IMAP Migration** | imap-sync |

ddashboard **kept** its own IMAP **import** into MySQL (`dls_mail_*`). That is not a mailbox copy. See [`email-indexing.md`](email-indexing.md) (embedding pipeline decommissioned) and [`../stella-server/imap-sync-service.md`](../stella-server/imap-sync-service.md).

---

## 7. Hosting sketch

```
Hetzner managed (foxcrr / nest)
├── ddashboard          WordPress theme + dls/v1
└── Foxx Cloud          Laravel on dedi6724 — calls Stella media-api

Hetzner Cloud  (same project as Edison)
└── atlas  CX33  2.31.42.13
    │     https://atlas.foxcraft.digital
    │     api.atlas.foxcraft.digital → atlas-api (127.0.0.1:8090), allowlist 2.31.42.13 + 95.217.144.93
    │     Laravel 12.61, PHP 8.5.4-FPM (Ubuntu 8.5.4-0ubuntu1.5, includes CVE-2026-109618/109619 backports), MySQL `atlas_prod`, Redis (cache)
    │     systemd atlas-queue.service  →  queue:work database --queue=deployments
    │     /etc/cron.d/atlas → schedule:run (health + WP uptime + drive-backups + hourly TT open sync)
    │
    ├── HTTPS ──► Stella (Hetzner dedicated, 95.217.144.93)
    │               Caddy :443  (DOCKER-USER ACCEPTs 2.31.42.13)
    │               ├── stella.foxcraft.digital/stella      → stella-api   (ddashboard)
    │               ├── stella.foxcraft.digital/imap-sync   → imap-sync    (Atlas)
    │               ├── stella.foxcraft.digital/ollama      → ollama
    │               ├── stella-deployment-api.foxcraft.digital → deploy-api (Atlas prod)
    │               ├── stella-health-api.foxcraft.digital     → health-api (Atlas prod + atlas.dev)
    │               ├── stella-media-api.foxcraft.digital      → media-api (Foxx Cloud / nest)
    │               └── atlas.dev.foxcraft.digital             → Atlas Laravel dev (port 2205 SSH)
    │
    ├── SSH ──► GitHub, rsync destinations (28 deploy_servers)
    └── HTTPS ──► registered WP sites (uptime + Atlas Connect)

Hetzner Cloud  (same project)
└── Edison              coding-agent host — Atlas reads Cloud backups/status only
```

---

## 8. Glossary

| Term | Meaning |
|------|---------|
| **ddashboard** | WordPress theme / business product |
| **Atlas** | Laravel ops platform at `atlas.foxcraft.digital` (old URL `dev.atlas.foxcraft.digital` still on the managed host until retired). Dev instance: `atlas.dev.foxcraft.digital` on Stella |
| **Atlas Connect** | WordPress plugin on each managed site; one-time admin login + encrypted S3 backups + infection scan (`atlas-connect/v1`). Site never phones home |
| **Stella** | Dedicated server hosting AI and ops services |
| **stella-api** | FastAPI app — `/chat/*` only |
| **deploy-api** | FastAPI app — signed deploy trigger + status |
| **health-api** | FastAPI app — signed system-status snapshot |
| **media-api** | FastAPI app — RAW/HEIC/JPEG/video-poster decode for Foxx Cloud (HMAC; `/convert` body or `/convert-remote` Storage Box). `https://stella-media-api.foxcraft.digital` |
| **Foxx Cloud** | Laravel app on the nest (`dedi6724`). Calls media-api. Not one of the four named Vegapunk systems. |
| **atlas-api / Atlas host API** | FastAPI on the Atlas host (`https://api.atlas.foxcraft.digital` → `127.0.0.1:8090`). Signed `system-health` snapshot of the Atlas box. See [`../atlas/host-api.md`](../atlas/host-api.md) |
| **imap-sync** | Express + `imapsync` mailbox copy |
| **Edison** | Separate coding-agent host |
| **`ddashboard` principal** | Name in Stella `allowed_signers` for Atlas’s SSH key (same keypair as the old managed-host Atlas; comment `atlas-prod`) |
