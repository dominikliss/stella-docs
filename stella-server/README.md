# Stella

Dedicated AI / services host. No general-purpose UI — **Atlas** and **ddashboard** are the HTTP clients.

**System map:** [`../integration/system-overview.md`](../integration/system-overview.md)

---

## Host and services

| Document | Contents |
|----------|----------|
| [infrastructure.md](infrastructure.md) | Servers, ports, Docker, Caddy, Ollama, firewall |
| [stella-api.md](stella-api.md) | FastAPI `/chat/*` (ddashboard) |
| [deploy-api.md](deploy-api.md) | Signed deploy trigger (Atlas) |
| [health-api.md](health-api.md) | Signed system-status snapshot (Atlas) |
| [imap-sync-service.md](imap-sync-service.md) | Express + `imapsync` mailbox copy (Atlas) |
| [backup.md](backup.md) | Nightly restic → Storage Box |
| [dev-ssh-access.md](dev-ssh-access.md) | Per-app SSH containers (not host users) |
| [client-ip-access.md](client-ip-access.md) | Per-subdomain client IP grants |
| [dotnet-app-deployment.md](dotnet-app-deployment.md) | .NET pattern: Stella dev containers → Azure production |

Atlas halves of the signed APIs: [`../atlas/deploy-pipeline.md`](../atlas/deploy-pipeline.md), [`../atlas/monitoring.md`](../atlas/monitoring.md), [`../atlas/tools.md`](../atlas/tools.md).

---

## Apps on this host

App-specific runbooks (not Stella-the-product) live under [`apps/`](apps/).

| App | Docs |
|-----|------|
| osgar-datahub | [`apps/osgar-datahub/`](apps/osgar-datahub/) |
| advoapp | Pattern in [dotnet-app-deployment.md](dotnet-app-deployment.md) (no separate folder yet) |
