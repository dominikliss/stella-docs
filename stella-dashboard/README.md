# stella-dashboard — documentation

WordPress **ddashboard** theme: product overview, deep dives, and **mirrors of Cursor agent rules** (for Notebook LM / humans). The live agent rules stay in the theme at `.cursor/rules/*.mdc`; substantive copies are under **`cursor-rules/*.md`**.

## Index

| File | Topic |
|------|--------|
| [architecture.md](architecture.md) | Full architecture (CPTs, REST `dls/v1`, mail, AI analyses, PM, KSeF, PDF, …) |
| [CAPABILITIES.md](CAPABILITIES.md) | Modules and capabilities overview |
| [mail-nachrichten.md](mail-nachrichten.md) | IMAP tables, REST, inbox UI |
| [mail-cron-hosting.md](mail-cron-hosting.md) | Cron / hosting notes for mail |
| [ai-chat.md](ai-chat.md) | AI chat notes |
| [marketing-youtube-plan.md](marketing-youtube-plan.md) | Marketing / YouTube |
| [PM_TRACKINGTIME_FIELD_MAP.md](PM_TRACKINGTIME_FIELD_MAP.md) | PM ↔ TrackingTime fields |
| [TRACKINGTIME_API_REFERENCE.md](TRACKINGTIME_API_REFERENCE.md) | TrackingTime API v4 (short) |
| [TRACKINGTIME_API_VERBATIM_EXPORT.md](TRACKINGTIME_API_VERBATIM_EXPORT.md) | TrackingTime API (verbatim) |
| [REFACTORING.md](REFACTORING.md) | Refactoring notes |
| [DESIGN_SYSTEM.md](DESIGN_SYSTEM.md) | Design system (legacy / long reference) |
| [STATS_PAGE_DESIGN_BRIEF.md](STATS_PAGE_DESIGN_BRIEF.md) | Stats page design brief |

## Integration with Stella

Not WordPress-only: email **vector indexing** and Stella API are documented under:

- [../integration/ddashboard-and-stella-server.md](../integration/ddashboard-and-stella-server.md)
- [../integration/email-indexing.md](../integration/email-indexing.md)

## Cursor rule exports

| Folder | Contents |
|--------|----------|
| [cursor-rules/](cursor-rules/) | One `.md` per `.mdc` in the theme (prefixed note + body) |
