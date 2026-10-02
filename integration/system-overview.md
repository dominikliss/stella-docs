# System overview — how the pieces connect

This is the **cross-system map** for foxcraft.digital: who owns what, which host talks to which, and which trust model sits on each path.

**Related docs**

- Atlas platform: [`../atlas/README.md`](../atlas/README.md)
- Atlas architecture: [`../atlas/architecture.md`](../atlas/architecture.md)
- Atlas ↔ ddashboard login: [`../atlas/auth.md`](../atlas/auth.md)
- Atlas ↔ WordPress sites: [`../atlas/atlas-connect.md`](../atlas/atlas-connect.md)
- ddashboard ↔ Stella (AI chat only): [`ddashboard-and-stella-server.md`](ddashboard-and-stella-server.md)
- Stella host: [`../stella-server/infrastructure.md`](../stella-server/infrastructure.md)
- Edison (coding-agent host): [`../edison/README.md`](../edison/README.md)

---

## 1. The four named systems

Servers use One Piece Vegapunk-satellite names. Firewalls, IPs, and networks use **functional** names.

| Name | What it is | Host | Owns |
|------|------------|------|------|
| **ddashboard** | Internal business app (CRM, accounting, mail, PM, AI chat) | Hetzner managed WordPress — same account as Atlas | Relational business data, WP sessions, `dls/v1` auth tokens |
| **Atlas** | Internal ops platform (deploys, Stella health, WP uptime, IMAP migration) | Hetzner managed Laravel — `https://dev.atlas.foxcraft.digital` | Deploy history, health history, WP-site registry, IMAP job history |
| **Stella** | Dedicated AI / services machine | Hetzner dedicated | Ollama, stella-api, deploy-api, health-api, imap-sync, Caddy, .NET dev apps |
| **Edison** | Coding-agent sandbox | Hetzner Cloud (separate machine) | Cursor CLI container, sshfs mounts. **Must not** run on Stella. Atlas does not call Edison today. |

Neither Atlas nor ddashboard replaces the other:

- **ddashboard** is the product the business runs on (clients, invoices, mail, projects).
- **Atlas** is the ops console that used to live as **Werkzeuge** inside ddashboard (deploy, WP sites, mailbox copy). Those screens were removed from the theme on 2026-09-02.

Stella is a **service host**. It does not have a general-purpose UI. Atlas and ddashboard are the two HTTP clients that matter.

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
        │                                                 │ 3. plain HTTPS (trusted network)
        │                                                 ▼
        │                                           Stella imap-sync
        │                                           WP sites (HEAD + Atlas Connect)
        │                                           GitHub (SSH clone)
        │                                           destination SSH servers (rsync)
        │
        └── browser never talks to Stella or to client WP sites directly
            (except opening an Atlas Connect one-time login URL)


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

1. The **browser never calls Stella**. Atlas and ddashboard proxy every Stella call server-side.
2. Atlas **does not** issue its own login passwords. Identity lives in ddashboard (`/dls/v1/auth/*`).
3. Stella **deploy-api** and **health-api** do not use bearer tokens. They verify an OpenSSH signature from Atlas’s SSH private key, principal `ddashboard` (shared key with the managed host).
4. Stella **stella-api** (`/chat/*`) and **imap-sync** have no HTTP auth — ingress is restricted at the network / Caddy layer.
5. Atlas itself is IP-allowlisted (`ATLAS_ALLOWED_IPS`) except `GET /api/health`.

---

## 3. What each HTTP path is for

### Atlas → ddashboard (identity)

| Atlas call | ddashboard route | Purpose |
|------------|------------------|---------|
| `POST /api/auth/login` | `POST /dls/v1/auth/login` | Exchange username/password for access + refresh tokens |
| `POST /api/auth/refresh` | `POST /dls/v1/auth/refresh` | Rotate access token |
| `POST /api/auth/logout` | `POST /dls/v1/auth/logout` | Revoke refresh token |
| Token middleware (cache miss) | `GET /dls/v1/auth/verify` | Validate bearer token, read `{ id, email, name, roles[] }` |

Details: [`../atlas/auth.md`](../atlas/auth.md).

### Atlas → Stella (ops)

| Atlas feature | Stella service | Auth | Contract |
|---------------|----------------|------|----------|
| Deploy type `stella_deploy_api` | **deploy-api** `https://stella-deployment-api.foxcraft.digital` | SSH signature, payload `deploy-{app}:{ts}` / `list-apps:{ts}` | [`../stella-server/deploy-api.md`](../stella-server/deploy-api.md), [`../atlas/deploy-pipeline.md`](../atlas/deploy-pipeline.md) |
| Monitoring cron + dashboard | **health-api** `https://stella-health-api.foxcraft.digital` | SSH signature, payload `system-health:{ts}` | [`../stella-server/health-api.md`](../stella-server/health-api.md), [`../atlas/monitoring.md`](../atlas/monitoring.md) |
| Tools → IMAP Migration | **imap-sync** `https://stella.foxcraft.digital/imap-sync` | Network restriction only | [`../stella-server/imap-sync-service.md`](../stella-server/imap-sync-service.md), [`../atlas/tools.md`](../atlas/tools.md) |
| Health-api **fallback** probes | Public Caddy hosts (`/chat/health`, `/ollama`, apps) | None | [`../atlas/monitoring.md`](../atlas/monitoring.md) |

Atlas **does not** call `stella-api` `/chat/*`. That path is ddashboard’s.

### ddashboard → Stella (AI)

| Feature | Stella service | Auth |
|---------|----------------|------|
| `general` agent streaming chat | **stella-api** `POST /chat/stream` | Network restriction |
| Other agents / mail analyses | Ollama (or Anthropic / OpenAI) | Network restriction / provider key |

Details: [`ddashboard-and-stella-server.md`](ddashboard-and-stella-server.md).

### Atlas → WordPress sites (client / own sites)

| Feature | Target | Auth |
|---------|--------|------|
| Uptime every 5 min | `HEAD https://{domain}` | None (TLS verify off) |
| One-click WP admin login | `POST https://{domain}/wp-json/dl-connect/v1/generate-token` | Shared **Atlas Connect** secret; Atlas host IP must be allowlisted on the site |

Details: [`../atlas/atlas-connect.md`](../atlas/atlas-connect.md).

### Atlas → GitHub + SSH servers (classic deploys)

| Feature | Target | Auth |
|---------|--------|------|
| `ssh_rsync` clone | `git@github.com:{owner}/{repo}.git` | Same Atlas SSH key as a GitHub deploy key |
| `ssh_rsync` sync | `user@host:path` | Same key in `authorized_keys` |
| Branch picker / SSH folder browser | GitHub via SSH; remote `ls` via SSH | Same key |

Details: [`../atlas/deploy-pipeline.md`](../atlas/deploy-pipeline.md).

---

## 4. Shared SSH key (the one secret that spans three jobs)

Atlas uses **one SSH private key** (Settings → GitHub, `app_settings.deploy_ssh_key_path`, fallback `DEPLOY_SSH_KEY_PATH`):

1. **GitHub** — clone private repos (`ssh_rsync`)
2. **Destination servers** — rsync
3. **Stella** — `ssh-keygen -Y sign -n deploy` for deploy-api and health-api

Stella’s `deploy-api/allowed_signers` lists this public key as principal **`ddashboard`**. health-api mounts the same file read-only. The principal name is historical: Atlas and ddashboard share the managed-host keypair, so no second signer was added when Atlas took over ops.

The private key never crosses the network. Only a short-lived signed payload (`action:unix_timestamp`, 60 s replay window on Stella) is posted.

---

## 5. Data ownership

| Data | Source of truth | Replica / history |
|------|-----------------|-------------------|
| Users, roles, passwords | **ddashboard** (WordPress) | Atlas `users` + Spatie roles (synced on login / verify) |
| Clients, invoices, mail, PM | **ddashboard** MySQL | — |
| Deploy configs, runs, logs | **Atlas** MySQL | Stella deploy-api keeps its own in-memory/job logs during a run |
| Stella health snapshots | **Atlas** (`stella_health_checks`, `stella_service_uptime_logs`) | health-api is live-only; it does not store history |
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
Hetzner managed (foxcrr)
├── ddashboard          WordPress theme + dls/v1
└── Atlas               Laravel 12 + Vite/React SPA
    │                   MySQL `atlas_dev`, Redis (cache), database queue `deployments`
    │                   cron → schedule:run (health + WP uptime)
    │                   cron + flock → scripts/queue-deployments.sh
    │
    ├── HTTPS ──► Stella (Hetzner dedicated)
    │               Caddy :443
    │               ├── stella.foxcraft.digital/stella      → stella-api   (ddashboard)
    │               ├── stella.foxcraft.digital/imap-sync   → imap-sync    (Atlas)
    │               ├── stella.foxcraft.digital/ollama      → ollama
    │               ├── stella-deployment-api.foxcraft.digital → deploy-api (Atlas)
    │               └── stella-health-api.foxcraft.digital     → health-api (Atlas)
    │
    ├── SSH ──► GitHub, rsync destinations
    └── HTTPS ──► registered WP sites (uptime + Atlas Connect)

Hetzner Cloud
└── Edison              coding-agent host (not on Atlas’s call graph today)
```

---

## 8. Glossary

| Term | Meaning |
|------|---------|
| **ddashboard** | WordPress theme / business product |
| **Atlas** | Laravel ops platform at `dev.atlas.foxcraft.digital` |
| **Atlas Connect** | WordPress plugin on each managed site; issues one-time admin login URLs (`dl-connect/v1`) |
| **Stella** | Dedicated server hosting AI and ops services |
| **stella-api** | FastAPI app — `/chat/*` only |
| **deploy-api** | FastAPI app — signed deploy trigger + status |
| **health-api** | FastAPI app — signed system-status snapshot |
| **imap-sync** | Express + `imapsync` mailbox copy |
| **Edison** | Separate coding-agent host |
| **`ddashboard` principal** | Name in Stella `allowed_signers` for the shared managed-host SSH key (used by Atlas) |
