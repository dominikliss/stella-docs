# Open Gaps & Next Steps

Last updated: 2026-11-04

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
  - Backups: Atlas driver, Settings vault, explorer, browser decrypt, daily `wp-sites:run-daily-backups` at 05:00 Vienna, and signed `.atlas.tar` download are in the Atlas tree ([`atlas/atlas-connect/backups.md`](atlas/atlas-connect/backups.md)). **TODO (Dominik):** confirm Settings → Backups bucket is `atlas-wp-sites-backups`, and that a `backups` (and `scans`) queue worker is running on prod / `atlas.dev`.
  - Infection scan: Atlas driver, inventory store, allowlist, verdict, alerts, and UI are built ([`atlas/atlas-connect/security-scan.md`](atlas/atlas-connect/security-scan.md#atlas-implementation)). **TODO (Dominik):** run the migration on `atlas.dev`, restart the backup queue worker so it also serves `scans`, add a `WP site security scan` notification rule, then review and trust each site’s first scan.
  - Atlas Connect hardening (2026-10-23): Atlas side done in the Atlas tree. Atlas never used the status routes, the S3 bucket and hosts are pinned in Settings and before every plugin call, errors are parsed as `error ?? message`, login `403` means the WP user is not an administrator, UI copy is updated, and IPv4 is forced. See [`atlas/atlas-connect/README.md` → Atlas work still to do](atlas/atlas-connect/README.md#atlas-work-still-to-do). **TODO (Dominik):** after deploying Atlas, open Settings → Backups and confirm the saved bucket is `atlas-wp-sites-backups` (an older value now blocks backups with a clear `422`). Make sure each site’s `wp_user` is an administrator.
  - **TODO (Dominik):** after deploying the plugin, test `generate-token` from Atlas prod **and** Atlas dev (Stella hairpin through Caddy may present a Docker IP in `X-Forwarded-For` → `401`), and open a login link from the office and from mobile data (only the office should work).
- [x] **Atlas TrackingTime import (PM in Atlas)** — Atlas Tools import + hourly open-task sync + Projects UI + one-way Trello push are in the Atlas tree. Trello: [`atlas/projects.md`](atlas/projects.md). ddashboard still must serve [`stella-dashboard/atlas-client-lookup.md`](stella-dashboard/atlas-client-lookup.md) (`GET /dls/v1/atlas/clients`, `/clients/{id}`, `/users`; Bearer + IP allowlist). Atlas (`2.31.42.13`) currently gets 401 from ddashboard — see production-host TODO above. Do not use `POST /dls/v1/pm/import-tracking-time-projects`.
- [x] **Optional: deploy trigger via API on the atlas host.** Live 2026-10-10 on the host API (`POST /system/deploy`, `atlas-dev` key only). See [`atlas/host-api.md`](atlas/host-api.md). Remaining work under Infrastructure.
- [x] **Planned: health-api for the atlas host.** Live 2026-10-10 as the Atlas host API (`https://api.atlas.foxcraft.digital`). See [`atlas/host-api.md`](atlas/host-api.md). Atlas-side client live (Infrastructure).

---

## Edison (coding-agent server)

Host rebuilt 2026-10-09 onto CX33 (`edison-ip` = `178.105.203.54`). See [`edison/README.md`](edison/README.md).

- [x] **Edison public IP confirmed** — closed 2026-10-09. Hetzner Primary IPv4 name is `edison-ip`, address `178.105.203.54` (moved from the old server). Stella `DOCKER-USER` already ACCEPTs it on 2201 and 2204; no Stella firewall change.
- [x] **Persist Cursor-agent SSH config / login state** — closed 2026-10-09. Host bind-mounts for `ssh-config`, `known_hosts`, `~/.cursor`, `~/.config/cursor`; systemd remounts sshfs at boot. See [`edison/cursor-agent.md`](edison/cursor-agent.md).
- [ ] **Split `authorized_keys` for `edison`** — done for hypershape (`atlas/keys/edison.pub`, 2026-11-04). Still shared on osgar and on `dominikliss` (legacy `Dockerfile.ssh`); moving `dominikliss` to the shared image closes that site. See [`stella-server/wordpress-staging.md`](stella-server/wordpress-staging.md) and [`stella-server/dev-ssh-access.md`](stella-server/dev-ssh-access.md).
- [ ] **Shared live tree / worktree** — agent sshfs-mounts the live osgar `src/`. Runs use `--force` on that tree. Guard text forbids `checkout` / `switch` / `stash` / `reset` / `commit`. **TODO (Dominik):** design a separate git worktree or clone.
- [ ] **`git safe.directory` for `edison` in `osgar-datahub-ssh`** — applied manually in that container; lost on rebuild. **TODO (Dominik):** bake into `Dockerfile.ssh`. The **agent-side** fix is closed: `cursor-agent` image sets `safe.directory '*'` (2026-11-02). See [`edison/cursor-agent.md`](edison/cursor-agent.md).
- [ ] **osgar `Dockerfile.ssh` supervisorctl password** — in plain text. **TODO (Dominik):** move to an env/secret and rotate. Do not write the value into docs.
- [x] **Default Cursor model** — closed 2026-11-02. Default is `auto`. Catalog is served by `GET /v1/capabilities` (~270 models). Ops pin via `EDISON_CURSOR_MODEL` when the client sends Auto. See [`edison/host-api.md`](edison/host-api.md).
- [ ] **`CURSOR_API_KEY` for unattended runs** — not set up yet. Third-party sources say unattended runs should use an API key.
- [x] **`dominikliss` sshfs mount** — closed 2026-11-02 via host-api `POST /v1/workspaces/ensure`. See [`edison/host-api.md`](edison/host-api.md).
- [x] **Web app to drive the agent (host side)** — closed 2026-11-01; run options 2026-11-02; workspaces v1 2026-11-02. `/v1` endpoints live in `host-api` (start / stream / abort / resume / capabilities / catalog / ensure / repos). `POST /v1/runs` accepts `mode`, `model`, `settings`. Workspace = catalog mount or a subdirectory. Cursor `stream-json` mapped. Session prefix is `cursor_`. Abort of unknown/finished session returns 404. See [`edison/host-api.md`](edison/host-api.md). Laravel Phase 2 + UI Phase 3 implemented in the Edison repo (2026-11-02): `RunOptions`, `GET /api/repos` capabilities, message fields, `chats.mode/model/settings`, searchable model picker. Laravel workspaces contract (catalog → ensure → repos → run on a leaf) is **not built yet**. **UNVERIFIED** on `edison.dev`. Live-tree item stays open.
- [ ] **Further agent images** — OpenCode as a second backend via that adapter; own Dockerfile, tool-named container, same `edison-net` and `/mnt/<full-hostname>` scheme.
- [x] **Edison firewall for HTTP/HTTPS** — closed 2026-11-01. TCP/443 opened, IP-restricted (`194.126.177.181`, `95.217.144.93`, `2.31.42.13`). Port 80 stays closed. See [`edison/infrastructure.md`](edison/infrastructure.md).
- [ ] **Atlas-side `CheckEdisonHealth` / `EdisonHealthRecorder`** — not built. Edison host API health is live; Atlas does not poll it yet. Extends existing `/monitoring/edison`. See [`atlas/monitoring.md`](atlas/monitoring.md).
- [ ] **Confirm Stella on Edison TCP/22 and ICMP** — live firewall allows `95.217.144.93` on 22 and ICMP, not only `194.126.177.181`. **TODO (Dominik):** confirm that is intentional.
- [ ] **Signed health call from Atlas prod** — **UNVERIFIED.** Stella + atlas-dev key works. Call from `2.31.42.13` with the atlas-prod key has not been run.
- [ ] **First-turn abort** — the app only learns `session_id` on `done`, so the first turn cannot be aborted. **TODO:** emit an early `session` SSE event. See [`edison/host-api.md`](edison/host-api.md).
- [ ] **advoapp.finditoo (2202)** — no `edison` user or key. Decide whether Edison should work there; if yes add user + `edison_agent` pubkey and rebuild `advoapp-ssh`. Until then `ensure` returns 422 (`Connection reset by peer`).
- [ ] **Full reboot test of the Edison box** — only `systemctl restart cursor-agent` was tested (2026-11-02). Mounts came back from `mounts.json`.
- [ ] **Atlas Edison card extra mounts** — **UNVERIFIED** whether `/monitoring/edison` renders extra mounts from `mounts.json`. **TODO (Dominik):** check.
- [ ] **Laravel workspaces contract** — switch to `GET /v1/workspaces` → `ensure` → `repos` → run on the leaf; key chats by `host + slug`; remove static `mounted` in `config/edison.php`; then drop `path` from the host response.
- [ ] **Drop the `EDISON_WORKSPACES` fallback** in `catalog.py` after one release.
- [ ] **No unmount API in v1** — removing a host is manual (`mounts.json` + `umount -l`).
- [ ] **Delete rollout backups** — `/opt/services/host-api/app.bak-2026-11-*` and `/root/cursor-agent/*.bak-2026-11-02` after a stable week.
- [ ] **Throwaway `*.staging.foxcraft.digital` sites** — `wp-staging-api` is superseded; staging is a stack-api `create` + `restore` (not built yet). Edison catalog/SSH for those sites still needs a spec: catalog rows per build, stable SSH host keys across rebuilds (otherwise `accept-new` fails with "host identification changed"), Atlas rebuild guard for dirty/unpushed repos. See [`stella-server/stack-api.md`](stella-server/stack-api.md).
- [ ] **sshfs error text** — a rejected key shows `Connection reset by peer` instead of `Permission denied`. Optional: run a `ssh -o BatchMode=yes … true` probe before sshfs to return a clearer 422.
- [ ] **UI end-to-end chat on `edison.dev`** — **UNVERIFIED.** Includes resume via the app. Browser test still open.
- [x] **Global agent skills on Edison** — closed 2026-11-03. Host sync (default-key deploy key, shallow clone, `sync-skills.sh`, timer, `.path` unit, `run.sh` hook) + `GET /v1/skills` / `POST /v1/skills/sync`. See [`edison/skills.md`](edison/skills.md).
- [ ] **`git-push-with-tag` is loaded globally on live staging trees.** Its description triggers on "push", "deploy", "ship", "release" and tells the agent to commit, tag and push — conflicts with the live-tree guard. Decide: allowlist in `sync-skills.sh`, or mark the skill IDE-only in `dev-docs`.
- [ ] **Deploy-key fingerprint** — **TODO (Dominik):** record in [`edison/skills.md`](edison/skills.md) (`ssh-keygen -lf ~/.ssh/id_ed25519.pub`). Do not invent it.
- [ ] **Skills UI: sync-now** — `/` autocomplete is in the Edison repo (calls `GET /v1/skills`). Optional Settings / "sync now" button (`POST /v1/skills/sync`) is not built. **UNVERIFIED** on `edison.dev`.
- [ ] **Live update test** — push to `dev-docs` `main`, confirm the host copy updates within 15 min. **UNVERIFIED.**

### Edison app (Laravel) — 2026-10-26

Repo `foxcraftdigital/edison`, dev instance `https://edison.dev.foxcraft.digital` on Stella (SSH 2207). See [`edison/app.md`](edison/app.md) and [`edison/dev-instance.md`](edison/dev-instance.md).

- [x] **Second Edison commit** — pushed 2026-10-26 as `acb37cc`, tag `0.1.1`.
- [ ] **Bump the stella-docs submodule pointer** in the Edison and Atlas repos after this docs update.
- [ ] **End-to-end ddashboard login on `edison.dev`** — **UNVERIFIED.** Includes whether ddashboard's IP allowlist accepts Stella (`95.217.144.93`); prod would also need `178.105.203.54`. Allowlist probably an `.htaccess` under `public_html` (same TODO as the Atlas 401/403 item above).
- [ ] **Clear error when `EDISON_DDASHBOARD_URL` is missing** — **TODO (Dominik):** make the driver throw "EDISON_DDASHBOARD_URL is not set" instead of `ConnectionException` "URI must include a scheme and host".
- [x] **Real agent link (host side)** — closed 2026-11-01; run options 2026-11-02; workspaces v1 2026-11-02. When `EDISON_HOST_API_URL` is set, the app calls `POST /v1/runs`. Host is live including `GET /v1/capabilities` and the workspaces endpoints. Laravel Phase 2 + UI Phase 3 are in the Edison repo (see [`edison/run-options.md`](edison/run-options.md)); Laravel workspaces contract still open above; worktree design (live-tree item above). **UNVERIFIED:** end-to-end chat on `edison.dev`.
- [ ] **Edison prod app** at `edison.foxcraft.digital`. Headless API (health + `/v1`, workspaces v1) is live at `api.edison.foxcraft.digital` (2026-11-01 / 2026-11-02). Atlas-side `CheckEdisonHealth` is not built.
- [ ] **`edison.dev` ssh container** — **TODO (Dominik):** decide on a home volume (git identity, `known_hosts`, clones are lost on rebuild); whether to split `authorized_keys` for `edison`; whether to add an editor (`nano`) to `Dockerfile.ssh`.
- [ ] **`edison.dev` in health-api and backup** — **UNVERIFIED** that the five containers are discovered.
- [ ] **Delete `~/atlas-ref`** in the `edison.dev` ssh container (read-only Atlas clone used during setup). **TODO (Dominik).**

---

## Email Indexing Pipeline — DECOMMISSIONED

> ChromaDB, `nomic-embed-text`, and all `/emails/*` Stella routes were removed on 2026-08-06. This gap is closed by removal, not implementation. See [`integration/email-indexing.md`](integration/email-indexing.md) for the full decommission notice and historical architecture.

---

## media-api (Foxx Cloud) — 2026-10-18

Service live at `https://stella-media-api.foxcraft.digital`. Contract: [`stella-server/media-api.md`](stella-server/media-api.md).

**Verified 2026-10-18:**
- [x] **Image build** — RawTherapee, exiftool, openssh-client included.
- [x] **`GET /health`** — `status` `ok`; tools include `rawtherapee-cli`, `exiftool`, `scp`, `sftp`; `remote.enabled` = true.
- [x] **ZV-E10 ARW through `POST /convert?op=master`** — `X-Render: rawtherapee`, `X-Profile: Sony ZV-E10 Camera Standard.dcp`, 6016×4016, about 10 s.
- [x] **`sftp` from inside the container** to the Storage Box (key, pinned `known_hosts`, port 23). Account home is `/home` (paths like `/home/travel/...`; `/travel/...` does not exist).

**Open / UNVERIFIED:**
- [ ] **Foxx Cloud PHP client** — MediaRaster using `/convert-remote` with `dest`; fallback to local convert; base URL and `MEDIA_HMAC_SECRET` in the nest’s `.env`. In progress.
- [ ] **`POST /convert-remote` end to end** — **UNVERIFIED:** fetch, render, write-back with `dest`. Being tested through the Foxx Cloud app. Do not mark verified until a successful test is confirmed.
- [ ] **Colour/detail vs RawTherapee 5.13 Mac export** — **UNVERIFIED.** Stella JPEG 6.8 MB vs Mac 5.2 MB; Debian’s RawTherapee is older.
- [ ] **Styles other than `standard`; camera bodies other than ZV-E10** — **UNVERIFIED.**
- [ ] **Real HEIC / DNG / CR3; `poster` / `preview` / `tile` with real files through the API** — **UNVERIFIED.**
- [ ] **Dedicated SSH key and Storage Box sub-accounts** — see Security in [`stella-server/media-api.md`](stella-server/media-api.md). **UNVERIFIED:** which other Storage Boxes (e.g. the restic backup box) the copied host key unlocks.
- [ ] **`bake` op** — apply look edits once MediaLookBaker’s input format is known. Not built.
- [ ] **Atlas dashboard** — **UNVERIFIED** whether Atlas shows media-api (container, http, cert) after its next poll.
- [ ] **2026-10-19 03:00 backup** — **UNVERIFIED.** Check `/opt/services/backup/last-run.json` for `status` success.
- [ ] **Bulk load** — **UNVERIFIED.** Watch `load_1m` vs Atlas thresholds during bulk RAW runs (Ollama cpuset `0-5`, media-api `6-11`, 2 workers).

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
- [ ] **WP staging: split `authorized_keys` for `edison`** — done for hypershape. `dominikliss` still copies one shared file; closed by moving it to the shared image. See [`stella-server/wordpress-staging.md`](stella-server/wordpress-staging.md).
- [ ] **WP staging: `FS_CHMOD_FILE` / `FS_CHMOD_DIR`** — residual risk (UNVERIFIED): WordPress updates that chmod files to `0644` may lower the ACL mask again. Possible mitigation not tested. **TODO (Dominik).** See [`stella-server/wordpress-staging.md`](stella-server/wordpress-staging.md).
- [ ] **WP staging: mails and MainWP identity on `dominikliss.foxcraft.digital`** — exact copy; `mailpoet`, `wp-mail-smtp`, and `mainwp-child` left active. Real emails can leave staging; `mainwp-child` carries the live site’s identity. **TODO (Dominik):** decide whether to neutralise them. See [`stella-server/wordpress-staging.md`](stella-server/wordpress-staging.md).
- [ ] **WP staging: leftovers after the 2026-10-11 live import** — `wordpress:cli` image on the Stella host; `/tmp/wp-cli.phar` inside the `wp` container (removal not confirmed); `~/relaunch_dominikliss_com.sql` in the `-ssh` container (110 MB); leftover zip/dump on the managed host (**UNVERIFIED**). **TODO (Dominik).** See [`stella-server/wordpress-staging.md`](stella-server/wordpress-staging.md).
- [ ] **WP staging: `/root/wp-before-php85.sql`** — 86 MB dump on Stella from the 2026-10-23 PHP 8.5 bump. **TODO (Dominik):** delete once `dominikliss.foxcraft.digital` is confirmed fine. See [`stella-server/wordpress-staging.md`](stella-server/wordpress-staging.md).
- [ ] **`dominikliss` `wordpress:php8.5-apache` is a moving tag** — `wp` now runs an untagged image (`sha256:ce31904f…`) because `docker compose pull wp` for hypershape moved the tag. Next `up -d wp` on `dominikliss` picks up the new image; check patch level with `php -v`. See [`stella-server/php-updates.md`](stella-server/php-updates.md).
- [ ] **Managed host `dedi6724` PHP version** — operator says updated after CVE-2026-109618/109619 (ddashboard, Foxx Cloud, `dominikliss.com` live); exact version **UNVERIFIED**. See [`stella-server/php-updates.md`](stella-server/php-updates.md).
- [ ] **`hypershape.foxcraft.digital`** — WP staging on Stella. PHP 8.5.12 (`wordpress:php8.5-apache` + shared `stella/ssh-php:8.5`, adopted 2026-11-04). SSH port **2206**, on `internal` (never `edge`). `edison` has its own key (`atlas/keys/edison.pub`, UID 1002). Still open: `DOCKER-USER` source IPs for 2206 not documented; Edison login from Edison after the switch **UNVERIFIED** (host keys unchanged via the `ssh-keys` volume); no `uploads.ini` mount; delete `/root/hypershape-before-adopt.sql` and `docker-compose.yml.bak-20261104`. **TODO (Dominik).** See [`stella-server/wordpress-staging.md`](stella-server/wordpress-staging.md).
- [ ] **`stella-media-api` Caddy block** — `propagation_delay` / `propagation_timeout` lines are duplicated in the live Caddyfile. Works; **TODO (Dominik):** clean up.
- [ ] **Hetzner Cloud project name for `foxcraft.digital`** — zone moved; ID still `567656`. **TODO (Dominik):** name of the new project (docs previously said `konsoleH`).
- [x] **Stella `DOCKER-USER` 443 ACCEPT for Atlas** — `2.31.42.13` added. After any `rm` + nano recreation of `/usr/local/bin/docker-user-firewall.sh` it must be `chmod 755`. Port **2205** added (Dominik’s IP, managed host, Edison). Port **2207** added 2026-10-26 for Edison dev (same three IPs); 2206 is hypershape. Next free SSH port: **2208**.
- [ ] **Caddy `propagation_delay` on `dominikliss`** — untested idea after the zone-move ACME delay. Live `dominikliss` block does not currently have it (**UNVERIFIED**). **TODO (Dominik):** `grep -n propagation_delay /opt/services/caddy/Caddyfile`. `stella-media-api` **does** have `propagation_delay 45s` + `propagation_timeout 3m` (verified 2026-10-18). See [`stella-server/infrastructure.md`](stella-server/infrastructure.md) Issue 2.
- [ ] **Stella backup — first unattended 03:00 run** with the 2026-10-10 dynamic script. Manual runs verified. See [`stella-server/backup.md`](stella-server/backup.md). Also **UNVERIFIED:** 2026-10-19 03:00 (first run that includes `media-api/`).
- [ ] **Stella full restore test** — runbook at [`stella-server/restore.md`](stella-server/restore.md). **UNVERIFIED** end-to-end. **TODO (Dominik):** test on a throwaway Hetzner Cloud server.
- [ ] **Off-server backup secrets** — restic password, Storage Box access, Hetzner API token, DNS records. **TODO (Dominik):** store them in the password manager. Do not write values into docs.
- [ ] **Backup failure email** — **UNVERIFIED** whether a failed Stella backup sends an email via Brevo (docs only describe emails for WP site `down`). **TODO (Dominik):** check whether the fake-failure test produced an email; if not, decide on an Atlas-side alert. See [`atlas/monitoring.md`](atlas/monitoring.md).
- [x] **Atlas client for the host API** — Laravel `CheckAtlasHealth`, `atlas_health_checks` / `atlas_service_uptime_logs`, dashboard card + `/monitoring/atlas`. See [`atlas/monitoring.md`](atlas/monitoring.md). Hairpin from Atlas to the public URL still open below.
- [x] **Atlas host-api.md** — runbook at [`atlas/host-api.md`](atlas/host-api.md) (health + deploy, 2026-10-10).
- [ ] **Atlas prod via the public API URL** — test `https://api.atlas.foxcraft.digital` from Atlas itself (hairpin risk), otherwise use `127.0.0.1:8090` locally.
- [x] **Hetzner Cloud API in Atlas** — read-only `HETZNER_CLOUD_TOKEN` lists atlas + edison, Cloud backups, snapshots, volumes. Dashboard Edison card + `/monitoring/edison`. See [`atlas/monitoring.md`](atlas/monitoring.md).
- [x] **Edison host API (health + `/v1`)** — live 2026-11-01 at `https://api.edison.foxcraft.digital`; run options 2026-11-02; workspaces v1 2026-11-02 (catalog, `ensure`, repo scan, workspace rule). Docker Caddy + host-api on `edison-net`. See [`edison/host-api.md`](edison/host-api.md). Atlas-side `CheckEdisonHealth` still open. Cloud API still covers power/backup until that exists.
- [ ] **Patch `/usr/local/bin/atlas-deploy`:** `queue:restart` instead of `systemctl restart atlas-queue`; reject refs starting with `-`
- [ ] **First real deploy through the host API** (new commits) and record the result in [`atlas/host-api.md`](atlas/host-api.md)
- [x] **Atlas dev app: deploy client + UI behind a flag, off on prod** — `ATLAS_SELF_DEPLOY_ENABLED` (404 when off). Button on the Server: atlas card; toasts on terminal status. See [`atlas/monitoring.md`](atlas/monitoring.md).

---

## stack-api (Atlas container management) — 2026-11-04

Live at `https://stella-stack-api.foxcraft.digital`. Contract: [`stella-server/stack-api.md`](stella-server/stack-api.md). Isolated `stack` network (`services_stack`); only Caddy and `stack-api` are on it. Docker socket mounted (root-equivalent until `apply` exists — list/restart only). `wp-staging-api` is superseded and will not be deployed.

**Verified 2026-11-04:**
- [x] **Auth skeleton** — Caddy IP allowlist (Atlas prod + Stella), app-layer `X-Forwarded-For` check (missing/fake → 403), `X-Stack-Token`, signed `POST /ping` (`principal: atlas-prod`), replay and bad-token 401.
- [x] **DNS + cert** — A record `stella-stack-api` → `95.217.144.93`; Let's Encrypt production on first attempt with `propagation_delay 45s` / `propagation_timeout 3m`.
- [x] **Generic SSH image** — `stella/ssh-php:8.5` built 2026-11-04; `hypershape` adopted by hand. See [`stella-server/stack-api.md`](stella-server/stack-api.md#shared-ssh-image-stellassh-phpversion).
- [x] **Audit log + `list` / `get` / `restart`** — Docker Engine API via httpx + socket. 6 stacks listed from Atlas prod; `/opt/services` invisible; hypershape get/restart verified; `DELETE FROM audit` blocked. Job table deferred to `restore`.

**Open:**
- [ ] **Atlas client** — `.env` has `STELLA_STACK_API_URL` / `STELLA_STACK_API_TOKEN`; code does not read them yet. Planned `stack_api_ssh_key_path` → `/home/atlas/.ssh/atlas-stack-api`.
- [ ] **Atlas-dev signer** — IP layer already passes from Stella; do **not** add principal `atlas-dev` until code restricts it to stacks marked `sandbox: true`.
- [ ] **Audit rejected requests** — `400` / `404` before the lookup are not written to `audit` yet.
- [ ] **Bind IP in `ports` output** — `127.0.0.1` vs `0.0.0.0` is not shown yet.
- [ ] **WP blueprint in stack-api** (then move `dominikliss`).
- [ ] **WP-CLI Deprecated notice on PHP 8.5** — decide whether to suppress `E_DEPRECATED` for the `wp` wrapper only.
- [ ] **`apply` / `restore` / `php-version` / `delete` / Adminer** — then Laravel and .NET blueprints.
- [ ] **Docker-socket policy checks** — refuse `privileged`, `cap_add`, `network_mode: host`, `pid: host`, bind mounts outside the app folder, `docker.sock`, `edge` on `ssh`/`db`/`adminer`, images outside the allowlist. See the planned section in [`stella-server/stack-api.md`](stella-server/stack-api.md).

---

## Closed 2026-11-04

- **stack-api auth skeleton** — live at `https://stella-stack-api.foxcraft.digital`. Isolated `stack` network, three protection layers (Caddy+app IP, token, SSH namespace `stack` with its own key). `GET /health` and signed `POST /ping` verified. No Docker socket yet. See [`stella-server/stack-api.md`](stella-server/stack-api.md).
- **`wp-staging-api` superseded** — will not be deployed as a separate service. Staging from an Atlas backup becomes a stack-api `create` + `restore` job.
- **Shared SSH image + hypershape migration** — `stella/ssh-php:8.5` (PHP 8.5.12, WP-CLI 2.12.0). Users from `atlas/users.conf` + per-user keys; persistent `/home`; `ssh` on `internal` only. hypershape moved off `php8.4` + per-site `Dockerfile.ssh`; `edison` has its own key. See [`stella-server/stack-api.md`](stella-server/stack-api.md#shared-ssh-image-stellassh-phpversion) and [`stella-server/wordpress-staging.md`](stella-server/wordpress-staging.md#ssh-container-shared-image).
- **stack-api Docker access** — Engine API over the socket (httpx). Signed `list` / `get` / `restart`; append-only `audit` table. `/opt/services` cannot be listed or restarted. See [`stella-server/stack-api.md`](stella-server/stack-api.md).
- **`advoapp-dev` 5080 and `osgar-datahub-dev` 5081** bound to `127.0.0.1` only (`docker port`, verified 2026-11-04).

## Closed 2026-11-03

- **Global agent skills on Edison** — deploy key (`id_ed25519`, comment `edison-dev-docs`), shallow `~/cursor-agent/dev-docs`, `sync-skills.sh`, timer + `.path` unit, `run.sh` hook, `GET /v1/skills` / `POST /v1/skills/sync`. Fingerprint, live-update test, and `git-push-with-tag` vs live-tree guard stay open. See [`edison/skills.md`](edison/skills.md).

## Closed 2026-11-02

- **Edison workspaces v1** — host-api catalog, `POST /v1/workspaces/ensure`, `GET /v1/workspaces/{host}/repos`, workspace rule for `/v1/runs`. Verified from Stella with the live bearer. See [`edison/host-api.md`](edison/host-api.md).
- **`dominikliss` sshfs mount** — created by `ensure`; remounted from `mounts.json` after `systemctl restart cursor-agent`.
- **`git safe.directory` on the agent side** — `cursor-agent` image sets `safe.directory '*'`. The osgar `-ssh` container item for user `edison` stays open.

## Closed 2026-11-01

- **Edison host API health** — `api.edison` DNS A record, Hetzner firewall TCP/443 (IP-restricted), Caddy (Let's Encrypt production, first attempt), `host-api` in `/opt/services`. `GET /health` and signed `POST /system/health` verified from Stella with the atlas-dev key. See [`edison/host-api.md`](edison/host-api.md).
- **Edison host API `/v1`** — same `host-api` process. Bearer token, Caddy `@v1_blocked`, SSE (`delta` / `done` / `error` / `: ping`), abort via `/usr/bin/kill`, sessions in `host-api-state`. Streaming run + resume + abort verified from the MacBook. `/v1/health` from the `edison.dev` app container ok. Atlas-side poller and UI e2e remain open.
- **Edison firewall TCP/443** — opened for `194.126.177.181`, `95.217.144.93`, `2.31.42.13`. Port 80 not opened.
- **Web app agent link (host side)** — `edison.dev` has `EDISON_HOST_API_URL` / token / `EDISON_AGENT=cursor` / `EDISON_AGENT_TIMEOUT=900`. App calls `POST /v1/runs` when the URL is set.

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
- Atlas Connect plugin: email module (SMTP override, dev catch-all for `wp_mail()`, test send) and further security features (login attempt log, IP blocklist, user audit log, admin 2FA). The empty stub classes were removed from the plugin on 2026-10-23.
