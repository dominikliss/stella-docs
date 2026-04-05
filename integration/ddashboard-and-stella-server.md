# ddashboard and Stella server — how they fit together

This document describes the **roles**, **network paths**, **data flows**, and **operational boundaries** between the **ddashboard** WordPress theme (Hetzner managed hosting) and the **Stella** stack (Hetzner dedicated server: FastAPI, Ollama, ChromaDB, Caddy, optional IMAP-sync service).

**Related docs**

- Server topology and ports: [`../stella-server/infrastructure.md`](../stella-server/infrastructure.md)
- Stella HTTP API (FastAPI): [`../stella-server/stella-api.md`](../stella-server/stella-api.md)
- Email embed queue end-to-end: [`email-indexing.md`](email-indexing.md)
- WordPress theme architecture (long): [`../stella-dashboard/architecture.md`](../stella-dashboard/architecture.md)

---

## 1. Roles

| System | Role |
|--------|------|
| **ddashboard** | Custom WordPress theme: CRM, accounting, IMAP mail storage in MySQL (`dls_*` tables), REST API `dls/v1`, React SPA, Anthropic-based AI chat and Ollama-based email analyses (jobs started from WP). **Source of truth** for email rows, clients, transactions, and configuration stored in WP options. |
| **Stella** | Dedicated AI / services host: **Ollama** (LLM + embeddings), **ChromaDB** (vector store v2 API), **stella-api** (FastAPI) exposing `/emails/upsert`, `/emails/query`, etc., **Caddy** reverse proxy on `:8080`. Optionally separate **imap-sync** container (not the same as ddashboard’s IMAP sync). |

Neither system replaces the other: WordPress owns relational data and user sessions; Stella owns **vector search** and can run **heavy models** close to Ollama/Chroma.

---

## 2. Network and trust model

- **Browser** talks only to **WordPress** (HTTPS). The browser **never** calls Stella directly for product features.
- **WordPress (ddashboard)** makes **server-to-server** HTTP(S) requests to Stella when features require it, for example:
  - **Email indexing:** `StellaEmailIndexClient` → `POST {base}/emails/upsert`
  - **Tests / health:** routes under `inc/routes/stella-api-test.php` (if configured)
- **Base URL and auth** for the index API are WordPress options (see `OptionService` / Verwaltung → AI-Anbindungen pattern):
  - `dls_stella_email_index_url` — base URL (trailing slash stripped in client)
  - `dls_stella_email_index_key` — optional; sent as **`X-Stella-Key`** when non-empty
  - `dls_email_embed_enabled`, `dls_email_embed_batch_size` — queue behaviour

Stella should be reachable from the **managed WordPress host’s outbound IP** (firewall / VPN rules are operator-specific; see Stella UFW notes in infrastructure doc). Treat the index key as a **shared secret** between the two servers, not as a substitute for network restriction.

---

## 3. Data flow — email indexing (high level)

```mermaid
flowchart LR
  subgraph wp [ddashboard_WordPress]
    IMAP[MailSync_IMAP]
    CRUD[EmailCrudService]
    Q[dls_email_embed_queue]
    CRON[dls_email_embed_process_cron]
    CLIENT[StellaEmailIndexClient]
  end
  subgraph stella [Stella_server]
    API[stella-api_FastAPI]
    OLL[Ollama_embed]
    CHR[ChromaDB]
  end
  IMAP --> CRUD
  CRUD --> Q
  CRON --> CLIENT
  CLIENT -->|POST_emails_upsert| API
  API --> OLL
  API --> CHR
```

1. Email rows persist in **`dls_email`** (MySQL) via existing IMAP pipeline.
2. When embed is enabled and the row is eligible, **`maybe_enqueue_email_embed()`** inserts/updates **`dls_email_embed_queue`**.
3. **WP-Cron** (`dls_email_embed_process_cron`, ~every 2 minutes) drains the queue and calls **`StellaEmailIndexClient::upsert()`**.
4. Payload includes stable document id **`email_{id}`**, plain-text document body (quote-stripped), and **metadata** (`sender`, `direction`, `date`, `subject`, `email_id` — verify against Stella `where` filters).
5. Stella embeds with **`nomic-embed-text`** and upserts into the **`emails`** Chroma collection.

**Detail and gaps:** [`email-indexing.md`](email-indexing.md).

---

## 4. AI features — who calls whom

| Feature | Where it runs | Typical backend |
|---------|----------------|-----------------|
| **AI chat (“Stella” / agents)** | Browser → WordPress `dls/v1/ai/chat` | **Anthropic** API (`AnthropicChatService`) — sessions in `dls_ai_chat_*` tables |
| **E-Mail-AI-Analysen** (writing style, classification) | WordPress async jobs / cron | **Ollama** on a configured host (Verwaltung), corpus from `dls_email` |
| **Vector search / RAG over mail** | WordPress → Stella | **`POST /emails/query`** on stella-api; Chroma similarity + metadata filters |
| **Email embedding / index** | WordPress → Stella | **`POST /emails/upsert`** |

So “Stella” as a **product name** can mean the **server** or the **assistant persona**; **embedding/query** paths are strictly **WordPress → stella-api → Chroma/Ollama**.

---

## 5. REST touchpoints (ddashboard)

| Area | Files (theme) |
|------|----------------|
| Queue + cron | `inc/email-embed-cron.php`, `inc/services/email-embed-queue-service.php` |
| HTTP client | `inc/services/stella-email-index-client.php` |
| Manual enqueue / REST | `inc/routes/mail-email-embed.php` |
| Health / query tests | `inc/routes/stella-api-test.php` |
| Options | `inc/services/option-service.php` (`dls_stella_*`, `dls_email_embed_*`) |

Paths are under WordPress REST prefix `/wp-json/dls/v1/...` (exact routes as registered in those files).

---

## 6. Security checklist

1. **Revoke any PAT** accidentally committed in `.gitmodules` or env files; use SSH or credential helper for **developer** clone of private `stella-docs`.
2. **Do not** expose Ollama/Chroma ports publicly without auth; follow Stella UFW/Caddy layout.
3. **Rotate** `dls_stella_email_index_key` if leaked; limit Stella ingress to known WP egress IPs where possible.
4. **HTML mail** stays sandboxed in the browser; only **sanitised/plain** text flows to Stella for embeddings (see client implementation).

---

## 7. Ops — submodule and documentation

- **Canonical documentation** for both worlds lives in the **`stella-docs`** Git repository (this tree), linked from the theme as **`docs/stella-docs`** submodule.
- When behaviour changes on either side, update:
  - **`stella-dashboard/`** for WordPress theme behaviour,
  - **`stella-server/`** for Docker/system/API on Stella,
  - **`integration/`** for cross-cutting flows (this file + email indexing).

**Bump submodule in theme repo:** commit inside `docs/stella-docs`, then in theme root: `git add docs/stella-docs` and commit the updated gitlink.

**Logs (Stella):** e.g. `docker logs services-stella-api-1 --tail 50` (container name may vary — see [`../stella-server/infrastructure.md`](../stella-server/infrastructure.md)).

---

## 8. Glossary

| Term | Meaning |
|------|---------|
| **ddashboard** | WordPress theme / product (internal business app) |
| **Stella** | Dedicated server hosting AI services |
| **stella-api** | FastAPI app on Stella (`/emails/*`, etc.) |
| **gitlink** | Git submodule pointer — parent repo stores commit SHA of `stella-docs` |
