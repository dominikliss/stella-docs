# Open Gaps & Next Steps

Last updated: 2026-10-16

**Cross-system overview:** [integration/system-overview.md](integration/system-overview.md) (Atlas + ddashboard + Stella + Edison). AI-chat path only: [integration/ddashboard-and-stella-server.md](integration/ddashboard-and-stella-server.md).

---

## Atlas documentation

- [x] **System-level Atlas docs** — closed 2026-10-01. Atlas is no longer only a folder of module notes. See [`atlas/README.md`](atlas/README.md), [`atlas/architecture.md`](atlas/architecture.md), [`atlas/auth.md`](atlas/auth.md), [`atlas/atlas-connect/`](atlas/atlas-connect/), [`atlas/tools.md`](atlas/tools.md), and [`integration/system-overview.md`](integration/system-overview.md). `atlas/dotnet-azure.md` now describes the live `stella_deploy_api` path (the old “post-deploy hook, not yet signed” plan is obsolete).
- [ ] **Atlas production host move (2026-10-10)** — live at `https://atlas.foxcraft.digital` on Hetzner Cloud CX33 `atlas` (`2.31.42.13`). Runbook: [`atlas/server.md`](atlas/server.md). Dev instance on Stella: [`atlas/dev-instance.md`](atlas/dev-instance.md). Open items:
  - **TODO (Dominik):** confirm `dev.atlas.foxcraft.digital` crons (`queue-deployments.sh`, `schedule:run`) are deactivated and decide when to retire the managed-host instance.
  - **TODO (Dominik):** remove `23.88.90.12` from `ATLAS_ALLOWED_IPS` unless something on the managed host really calls Atlas; reason for each entry unverified.
  - **TODO (Dominik):** bump `@types/node` to `^22` in the Atlas repo (`npm ci` currently needs `--legacy-peer-deps`).
  - **TODO (Dominik):** name of the Hetzner Cloud project that holds Atlas and Edison.
  - **TODO (Dominik):** rotate the reused `atlas-prod` SSH key (managed host still holds a copy). Deferred.
  - **TODO (Dominik):** test deployments from `atlas.dev` against one throwaway target (untested by design today).
  - **TODO (Dominik):** Atlas dev GitHub key is an account-level SSH key on `dominikliss` (reaches every repo that account can read). Consider a read-only deploy key. A customer deploy from `atlas.dev` succeeded on 2026-10-11 to the live `dominikliss.com` theme path; **UNVERIFIED** why `atlas-dev` could reach that host (docs still say it is in no `authorized_keys`). Check which `deploy_servers` entry and key were used. See [`atlas/dev-instance.md`](atlas/dev-instance.md).
  - **TODO (Dominik):** Atlas (`2.31.42.13`) gets 401 from ddashboard; Stella (`95.217.144.93`) gets 403 from the managed host’s Apache. Find the IP allowlist (probably `.htaccess` under `public_html`) and add Stella if Stella should call ddashboard. Required before Atlas can call the planned `/dls/v1/atlas/*` lookup routes.
  - Backups: `wp-sites:drive-backups` is scheduled every minute; backup driver not finished.
- [ ] **Atlas TrackingTime import (PM in Atlas)** — planned 2026-10-16. One-way import of TT projects (lists, tasks, subtasks, attachments, time entries) into Atlas MySQL. ddashboard keeps CRM. **ddashboard first:** implement [`stella-dashboard/atlas-client-lookup.md`](stella-dashboard/atlas-client-lookup.md) (`GET /dls/v1/atlas/clients`, `/clients/{id}`, `/users`; Bearer + IP allowlist `2.31.42.13` and `95.217.144.93` only). Then Atlas Tools wizard. Do not use `POST /dls/v1/pm/import-tracking-time-projects`.
- [x] **Optional: deploy trigger via API on the atlas host.** Live 2026-10-10 on the host API (`POST /system/deploy`, `atlas-dev` key only). See [`atlas/host-api.md`](atlas/host-api.md). Remaining work under Infrastructure.
- [x] **Planned: health-api for the atlas host.** Live 2026-10-10 as the Atlas host API (`https://api.atlas.foxcraft.digital`). See [`atlas/host-api.md`](atlas/host-api.md). Atlas-side client live (Infrastructure).

---

## Edison (coding-agent server)

Host rebuilt 2026-10-09 onto CX33 (`edison-ip` = `178.105.203.54`). See [`edison/README.md`](edison/README.md).

- [x] **Edison public IP confirmed** — closed 2026-10-09. Hetzner Primary IPv4 name is `edison-ip`, address `178.105.203.54` (moved from the old server). Stella `DOCKER-USER` already ACCEPTs it on 2201 and 2204; no Stella firewall change.
- [x] **Persist Cursor-agent SSH config / login state** — closed 2026-10-09. Host bind-mounts for `ssh-config`, `known_hosts`, `~/.cursor`, `~/.config/cursor`; systemd remounts sshfs at boot. See [`edison/cursor-agent.md`](edison/cursor-agent.md).
- [ ] **Split `authorized_keys` for `edison`** — both `Dockerfile.ssh` files (osgar and dominikliss) still `COPY` one shared file to `dominik`, `pawel`, and `edison`. **TODO (Dominik):** whether to split. See [`stella-server/dev-ssh-access.md`](stella-server/dev-ssh-access.md).
- [ ] **Shared live tree / worktree** — agent sshfs-mounts the live osgar `src/`. Must not `checkout` / `switch` / `stash` / `reset` / `commit`. **TODO (Dominik):** design a separate git worktree or clone.
- [ ] **`git safe.directory` for `edison`** — applied manually in `osgar-datahub-ssh`; lost on rebuild. **TODO (Dominik):** bake into `Dockerfile.ssh`.
- [ ] **osgar `Dockerfile.ssh` supervisorctl password** — in plain text. **TODO (Dominik):** move to an env/secret and rotate. Do not write the value into docs.
- [ ] **Default Cursor model** — UNKNOWN. **TODO (Dominik):** `cursor-agent models` / `--help`, set `--model` for reproducible runs.
- [ ] **`CURSOR_API_KEY` for unattended runs** — not set up yet. Third-party sources say unattended runs should use an API key.
- [ ] **`dominikliss` sshfs mount** — SSH config entry exists (port 2204); no mount yet.
- [ ] **Web app to drive the agent** — adapter layer (start / stream / abort / resume) on `edison-net`, so OpenCode (`opencode serve`) can be a second backend later. Audit trail once runs are unattended.
- [ ] **Further agent images** — OpenCode as a second backend via that adapter; own Dockerfile, tool-named container, same `edison-net` and `/mnt/<full-hostname>` scheme.
- [ ] **Edison firewall for HTTP/HTTPS** — open only when the web app is exposed; keep IP-restricted. Internal container traffic does not need these rules.

---

## Email Indexing Pipeline — DECOMMISSIONED

> ChromaDB, `nomic-embed-text`, and all `/emails/*` Stella routes were removed on 2026-08-06. This gap is closed by removal, not implementation. See [`integration/email-indexing.md`](integration/email-indexing.md) for the full decommission notice and historical architecture.

---

## Infrastructure

- [ ] **GitHub migration** — move all repos from Bitbucket to GitHub
  - ddashboard
  - stella-api (currently not in version control — needs a repo)
  - finditoo (5 consumer repos + contracts)
- [ ] **stella-api in git** — `/opt/services/stella-api` is not version controlled yet
- [ ] **Daily connectivity tests** — automated login/fetch/update tests for contract sync with failure notifications (stack not finalized)
- [ ] **Atlas health-api: render newer payload keys** — ~~`docker_daemon`, `ollama_resource_usage`, `cpu_ram` (added later 2026-08-10). Polling + dashboard already live against `STELLA_HEALTH_API_URL`. See [`stella-server/health-api.md`](stella-server/health-api.md).~~ **Closed 2026-09-03.** All keys are now normalized by `StellaHealthApiClient::normalizeServices()` and stored as per-service `StellaServiceUptimeLog` rows. See [`atlas/monitoring.md`](atlas/monitoring.md).
- [ ] **Remove now-unneeded UFW rules for port 11434** — Ollama no longer needs direct host-level access now that it's fully behind `edge`/Caddy. Rules for the two static-IP whitelist entries can be deleted once the retired systemd `ollama.service` is confirmed not coming back.
- [ ] **Old host-side Ollama model files** — can be deleted once `docker exec ollama ollama list` confirms all 13 models are present in the new container volume. Not yet verified as of 2026-08-10 (pull was still in progress — 10/13 confirmed present at last check).
- [ ] **Remove dead `DOCKER-USER` rules for ports 3001 and 8080** — those publishes were deleted 2026-09-01 (`imap-sync` via Caddy only; Caddy `:8080` already unused). Rules kept as harmless leftovers. See [`stella-server/infrastructure.md`](stella-server/infrastructure.md).
- [ ] **`advoapp-dev` stuck container / orphaned `dotnet` chain** — same incident twice (2026-10-09, 2026-10-12). A dead `dotnet watch` left `dotnet run` → `finditoo.advoapp` children on PID 1 holding port 8080; `stopasgroup`/`killasgroup` did not prevent it. **TODO:** find what kills the original `dotnet watch`; consider a watchdog or health check for duplicate `dotnet` chains (container-level health-api cannot see this). Runbook: [`stella-server/dotnet-app-deployment.md`](stella-server/dotnet-app-deployment.md).
- [ ] **`advoapp-ssh` — Node.js and supervisord not yet evaluated** — host-key volume fix already applied; Node and the supervisord restart-control pattern (implemented for osgar-datahub only) have not been evaluated for `advoapp-ssh`. Apply from [`stella-server/apps/osgar-datahub/setup.md`](stella-server/apps/osgar-datahub/setup.md) if SCSS builds or SSH-triggered restarts are needed there.
- [ ] **osgar-datahub `supervisorctl` credentials** — plaintext password baked into both Dockerfiles' build args, including `Dockerfile.ssh`. **TODO (Dominik):** move to an env/secret and rotate. Do not write the value into docs. Acceptable for the current dummy-dev-DB environment, not suitable for any app with production data in `src/`.
- [ ] **osgar-datahub `DataProtection` key-ring warning** — `No XML encryptor configured` on every fresh run; cosmetic for dev, worth fixing before reusing this pattern for anything closer to production.
- [ ] **Client IP grant for `osgar.datahub.foxcraft.digital` — temporary** — IPs `213.47.151.242` and `89.67.29.69` added 2026-09-03; remove when client no longer needs access. See [`stella-server/client-ip-access.md`](stella-server/client-ip-access.md).
- [ ] **`49.13.27.117` on port 443** — ACCEPTed in `DOCKER-USER`; Caddy denies it on `stella`, `stella-deployment-api`, `stella-health-api`, and `dominikliss`, but **not** on `advoapp.finditoo`. **TODO (Dominik):** owner / expiry, and whether the `advoapp.finditoo` block should deny it too. See [`stella-server/client-ip-access.md`](stella-server/client-ip-access.md).
- [ ] **Port 8081 in `docker-user-firewall.sh`** — ACCEPT for the two team IPs, then DROP. Not documented anywhere else. **TODO (Dominik):** say what 8081 is, or mark it as unknown.
- [ ] **WP staging: split `authorized_keys` for `edison`** — `dominikliss.foxcraft.digital` copies one shared file to all three users (same as osgar today), although [`stella-server/dev-ssh-access.md`](stella-server/dev-ssh-access.md) says `edison` should have its own key file. **TODO (Dominik):** whether to split. See [`stella-server/wordpress-staging.md`](stella-server/wordpress-staging.md).
- [ ] **WP staging: `FS_CHMOD_FILE` / `FS_CHMOD_DIR`** — residual risk (UNVERIFIED): WordPress updates that chmod files to `0644` may lower the ACL mask again. Possible mitigation not tested. **TODO (Dominik).** See [`stella-server/wordpress-staging.md`](stella-server/wordpress-staging.md).
- [ ] **WP staging: mails and MainWP identity on `dominikliss.foxcraft.digital`** — exact copy; `mailpoet`, `wp-mail-smtp`, and `mainwp-child` left active. Real emails can leave staging; `mainwp-child` carries the live site’s identity. **TODO (Dominik):** decide whether to neutralise them. See [`stella-server/wordpress-staging.md`](stella-server/wordpress-staging.md).
- [ ] **WP staging: leftovers after the 2026-10-11 live import** — `wordpress:cli` image on the Stella host; `/tmp/wp-cli.phar` inside the `wp` container (removal not confirmed); `~/relaunch_dominikliss_com.sql` in the `-ssh` container (110 MB); leftover zip/dump on the managed host (**UNVERIFIED**). **TODO (Dominik).** See [`stella-server/wordpress-staging.md`](stella-server/wordpress-staging.md).
- [ ] **Hetzner Cloud project name for `foxcraft.digital`** — zone moved; ID still `567656`. **TODO (Dominik):** name of the new project (docs previously said `konsoleH`).
- [x] **Stella `DOCKER-USER` 443 ACCEPT for Atlas** — `2.31.42.13` added. After any `rm` + nano recreation of `/usr/local/bin/docker-user-firewall.sh` it must be `chmod 755`. Port **2205** added (Dominik’s IP, managed host, Edison). Next free SSH port: **2206**.
- [ ] **Caddy `propagation_delay` on `dominikliss`** — untested idea after the zone-move ACME delay. Live block does not currently have it (**UNVERIFIED**). Result of `grep -n propagation_delay /opt/services/caddy/Caddyfile` on Stella: **TODO (Dominik)**.
- [ ] **Stella backup — first unattended 03:00 run** with the 2026-10-10 dynamic script. Manual runs verified. See [`stella-server/backup.md`](stella-server/backup.md).
- [ ] **Stella full restore test** — runbook at [`stella-server/restore.md`](stella-server/restore.md). **UNVERIFIED** end-to-end. **TODO (Dominik):** test on a throwaway Hetzner Cloud server.
- [ ] **Off-server backup secrets** — restic password, Storage Box access, Hetzner API token, DNS records. **TODO (Dominik):** store them in the password manager. Do not write values into docs.
- [ ] **Backup failure email** — **UNVERIFIED** whether a failed Stella backup sends an email via Brevo (docs only describe emails for WP site `down`). **TODO (Dominik):** check whether the fake-failure test produced an email; if not, decide on an Atlas-side alert. See [`atlas/monitoring.md`](atlas/monitoring.md).
- [x] **Atlas client for the host API** — Laravel `CheckAtlasHealth`, `atlas_health_checks` / `atlas_service_uptime_logs`, dashboard card + `/monitoring/atlas`. See [`atlas/monitoring.md`](atlas/monitoring.md). Hairpin from Atlas to the public URL still open below.
- [x] **Atlas host-api.md** — runbook at [`atlas/host-api.md`](atlas/host-api.md) (health + deploy, 2026-10-10).
- [ ] **Atlas prod via the public API URL** — test `https://api.atlas.foxcraft.digital` from Atlas itself (hairpin risk), otherwise use `127.0.0.1:8090` locally.
- [x] **Hetzner Cloud API in Atlas** — read-only `HETZNER_CLOUD_TOKEN` lists atlas + edison, Cloud backups, snapshots, volumes. Dashboard Edison card + `/monitoring/edison`. See [`atlas/monitoring.md`](atlas/monitoring.md).
- [ ] **Edison host API** — same pattern as Atlas (hostname `api.edison.foxcraft.digital` planned), not started. Cloud API covers power/backup status until then.
- [ ] **Patch `/usr/local/bin/atlas-deploy`:** `queue:restart` instead of `systemctl restart atlas-queue`; reject refs starting with `-`
- [ ] **First real deploy through the host API** (new commits) and record the result in [`atlas/host-api.md`](atlas/host-api.md)
- [x] **Atlas dev app: deploy client + UI behind a flag, off on prod** — `ATLAS_SELF_DEPLOY_ENABLED` (404 when off). Button on the Server: atlas card; toasts on terminal status. See [`atlas/monitoring.md`](atlas/monitoring.md).

---

## Closed 2026-10-10

- **Atlas backup `problems` + app-type grouping** — dashboard and `/monitoring/stella` show backup details (status, size, dumps, verify, problems) and group hosted containers as Dotnet / Laravel / WordPress. See [`atlas/monitoring.md`](atlas/monitoring.md).
- **health-api dynamic containers + certs** — no hardcoded app/cert list. `CORE_CONTAINERS` + `docker ps -a` + `seen.json`; certs parsed from the live Caddyfile. Verified: 20 containers up, 7 certs ok. See [`stella-server/health-api.md`](stella-server/health-api.md).
- **Backup automatic scope** — whole `/opt/services` + `/opt/apps`, discovered Docker volumes (including `caddy-data`), MariaDB/MySQL dumps, host config, weekly `restic check`. Labels are opt-out / extras, not the only scope. See [`stella-server/backup.md`](stella-server/backup.md).
- **`advoapp-redesign.finditoo.foxcraft.digital` removed from Stella** — containers `advoapp-redesign-dev` and `advoapp-redesign-ssh` stopped and removed; folder deleted; Caddy block removed; DNS A record `advoapp-redesign.finditoo` deleted in zone 567656; port-2203 `DOCKER-USER` rules removed; Cursor SSH config entry removed on Dominik's machine. Port 2203 is free; port 5082 is free again. Port-443 ACCEPT rules unchanged (port-wide, not tied to this app).
- **health-api own `allowed_signers`** — no longer reuses deploy-api’s file. Two `ddashboard` lines (`atlas-prod`, `atlas-dev`). See [`stella-server/health-api.md`](stella-server/health-api.md).
- **Atlas dev instance on Stella** — `https://atlas.dev.foxcraft.digital`, SSH port 2205. See [`atlas/dev-instance.md`](atlas/dev-instance.md).

## Closed 2026-10-09

- **Edison rebuilt on CX33** — old CPX32 (`edison-old`) deleted; IPv4 `edison-ip` `178.105.203.54` moved; Stella `DOCKER-USER` unchanged. New `edison_agent` key; ssh config / known_hosts / Cursor state persisted on the host; systemd remounts sshfs at boot. See [`edison/README.md`](edison/README.md).

## Closed 2026-10-03

- **Old Hetzner API token revoked** — done 2026-10-03. Zone `foxcraft.digital` moved to another Cloud project; replacement token is in `/opt/services/caddy/.env`. Token values are not recorded here.
- **Caddy `.env` backup deleted** — done 2026-10-03.

## Closed 2026-10-01

- **Atlas in the system docs** — `integration/system-overview.md` plus Atlas architecture / auth / Atlas Connect / tools. `atlas/dotnet-azure.md` rewritten onto the live `stella_deploy_api` path. health-api “What Atlas still needs” marked implemented.

## Closed 2026-09-03

- **Atlas-side health polling** — `CheckStellaHealth` job runs every 5 min; stores `StellaHealthCheck` + per-service `StellaServiceUptimeLog` rows. Signed `system-health` request to `STELLA_HEALTH_API_URL`, with public-probe fallback. See [`atlas/monitoring.md`](atlas/monitoring.md).
- **Atlas newer health-api payload keys** — `docker_daemon`, `ollama_resource_usage`, `cpu_ram`, `backup`, `backup_history` all normalized and stored. CPU/RAM/disk thresholds applied for aggregate status.
- **WP Sites uptime monitoring** — `CheckWpSiteUptime` runs every 5 min per site; stores `WpSiteUptimeLog`; sends Brevo email alert on first `down` transition per outage. See [`atlas/monitoring.md`](atlas/monitoring.md).
- **Stella deploy-api integration in Atlas** — new `stella_deploy_api` deploy type in Atlas: signed trigger, job-ID persistence for worker-restart recovery, live log streaming from Stella. See [`atlas/deploy-pipeline.md`](atlas/deploy-pipeline.md).
- **Atlas IMAP migration UI** — Atlas-side Tools → IMAP Migration proxies to the Stella `imap-sync` service. Stored `ImapMigrationJob` records in Atlas for history. See [`stella-server/imap-sync-service.md`](stella-server/imap-sync-service.md).
- **Atlas settings module** — GitHub SSH key path and Brevo transactional email (API key, from address, alert recipient) configurable in Settings UI, stored in `app_settings` table.

---

## Closed 2026-09-01

- **User-based SSH access on Stella** — implemented as per-app `-ssh` containers (`osgar-datahub` :2201, `advoapp` :2202) instead of host Linux users. Isolation is filesystem + Compose-project network, not per-dev identity inside one app. Root SSH remains Dominik-only. See [`stella-server/dev-ssh-access.md`](stella-server/dev-ssh-access.md).
- **Retired leftover Docker port publishes** — `imap-sync` `3001:3001` and Caddy `8080:8080` removed. Caddy now has a persistent `caddy-data` volume so ACME certs survive recreate.

---

## Closed 2026-08-10

- **`advoapp-dev` outage** — down 3 days with no monitoring in place. Root cause: no `restart` policy, killed by an unrelated Docker daemon restart on 2026-08-07. Fixed: `restart: unless-stopped` added to both `advoapp-dev` and `osgar-datahub-dev` (same gap, same template). See [`stella-server/dotnet-app-deployment.md`](stella-server/dotnet-app-deployment.md).
- **Missing Caddyfile `dir` pins** — 3 of 4 site blocks (`stella`, `advoapp.finditoo`, `stella-deployment-api`) were missing the Let's Encrypt production pin despite `infrastructure.md` claiming they had it. Only `osgar.datahub` actually had it. Fixed across all four. See [`stella-server/infrastructure.md`](stella-server/infrastructure.md) Issue 3.
- **Ollama gateway-IP routing inconsistency** — previously an open follow-up. Resolved by containerizing Ollama and switching Caddy's route to `ollama:11434` container-name routing. See [`stella-server/infrastructure.md`](stella-server/infrastructure.md) "Ollama" section.
- **Ollama had no enforced CPU limit** — documented `CPUQuota`/`AllowedCPUs` systemd settings were never actually applied (confirmed empty via `systemctl cat`). Now enforced at the Docker level (`cpus: "6"`, `cpuset: "0-5"`) as part of containerization.
- **Unused `advoapp` staging container** — built during initial setup, never routed, silently sat unused. Removed.
- **19GB of stale Docker build cache, an orphaned `/opt/github-runners` folder** — cleaned up.
- **`health-api` public subdomain** — `stella-health-api.foxcraft.digital` via Caddy (Atlas cannot reach edge-internal hosts). Signature still required for `/system/health`. See [`stella-server/health-api.md`](stella-server/health-api.md).
- **Atlas-side health polling** — ~~scheduled job + DB history + dashboard status grid against `STELLA_HEALTH_API_URL`~~ **(remaining gap: render newer payload keys above).** Closed 2026-09-03.
- **Ollama `OLLAMA_KEEP_ALIVE` changed from `-1` to `30m`** — the initial containerization set models to never unload, mirroring the old "preloaded permanently" systemd framing. Reconsidered same day: judged not worth holding 40GB+ RAM resident indefinitely regardless of actual use, especially now under a hard CPU cap shared with the rest of the box. `llama3.3:70b` and `phi4-mini` explicitly unloaded (`ollama stop`); no model is now treated as always-warm by default.

---

## Deferred (not in current scope)

- PM task indexing into a vector store (if rebuilt in the future)
- Codebase/Bitbucket indexing
- Action Scheduler (using WP-Cron for now)
- n8n workflows for email/Slack message-to-task extraction
- Fine-tuning `qwen2.5:14b` on writing style (ruled out — using prompt-based style profiles instead)
- KSeF-compliant automated accounting entries via Stella
- Slack assistant for client Q&A
