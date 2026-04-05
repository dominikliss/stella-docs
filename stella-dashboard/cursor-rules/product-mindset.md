# Cursor rule export: `product-mindset.mdc`

_Source of agent rule in theme repo: `.cursor/rules/product-mindset.mdc`._

---

# Product mindset (future sellable product)

This project may become a **product** for others. Prefer patterns that survive **multiple deployments, operators, and tenants** — not shortcuts that only work for one known server or one owner.

## Do

- **Configuration over hardcoding** — URLs, limits, feature toggles, credentials, and "who is the operator" belong in options, env, or install-time config — not scattered literals that assume this one domain.
- **Safe defaults** — Auth checks on new REST routes; validate and sanitise input; fail closed where security matters.
- **Clear boundaries** — Services and routes with explicit responsibilities; avoid "god" components or cross-cutting hacks that only the original author understands.
- **User-respectful UX** — Meaningful errors (German copy where the app is German), loading states, and recoverable flows — not silent failures or `console`-only diagnostics for real users.
- **Extensibility** — Hooks, filters, or small extension points where WordPress allows; avoid blocking forks by baking in unmaintainable one-off behaviour.
- **Observability without leaking data** — Structured logs or status responses that help operators; never log or echo sensitive payloads (see `no-db-data-access.mdc`).

## Avoid (typical in-house traps)

- "We'll only ever run this on our box" — **no** undocumented server quirks, IP lists without explanation, or magic constants that encode a single business.
- "I'll fix it in production" — **no** brittle assumptions about paths, PHP extensions, or cron timing without guards or docs in code comments _why_ when non-obvious.
- **Owner-only mental model** for anything that might ship — multi-user and permission paths should stay plausible even if v1 is single-tenant.

## AI layer

- All AI/LLM calls are **async only** — never block an HTTP request waiting for inference (use Action Scheduler).
- Layer 1 is always a deterministic rule-based classifier before touching a model. Skip it only if explicitly justified.
- ChromaDB is a **search index**, not a primary store — MySQL is source of truth. Use ChromaDB v2 API only.

## Relation to existing rules

This does **not** override architecture, security, or locale rules — it **amplifies** them for long-term maintainability and resale. When in doubt, choose the option that is easier for a **new team** or **new install** to run safely.
