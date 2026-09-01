# Open Gaps & Next Steps

Last updated: 2026-09-01

**Cross-system overview:** [integration/ddashboard-and-stella-server.md](integration/ddashboard-and-stella-server.md)

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
- [ ] **Atlas health-api: render newer payload keys** — `docker_daemon`, `ollama_resource_usage`, `cpu_ram` (added later 2026-08-10). Polling + dashboard already live against `STELLA_HEALTH_API_URL`. See [`stella-server/health-api.md`](stella-server/health-api.md).
- [ ] **Remove now-unneeded UFW rules for port 11434** — Ollama no longer needs direct host-level access now that it's fully behind `edge`/Caddy. Rules for the two static-IP whitelist entries can be deleted once the retired systemd `ollama.service` is confirmed not coming back.
- [ ] **Old host-side Ollama model files** — can be deleted once `docker exec ollama ollama list` confirms all 13 models are present in the new container volume. Not yet verified as of 2026-08-10 (pull was still in progress — 10/13 confirmed present at last check).
- [ ] **Remove dead `DOCKER-USER` rules for ports 3001 and 8080** — those publishes were deleted 2026-09-01 (`imap-sync` via Caddy only; Caddy `:8080` already unused). Rules kept as harmless leftovers. See [`stella-server/infrastructure.md`](stella-server/infrastructure.md).

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
- **Atlas-side health polling** — scheduled job + DB history + dashboard status grid against `STELLA_HEALTH_API_URL` (remaining gap: render newer payload keys above).
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
