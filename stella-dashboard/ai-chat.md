# AI chat (Claude) in ddashboard

## Setup

1. **Anthropic API key** — [Console](https://console.anthropic.com/) → create an API key.
2. **Verwaltung** (site) → section **Claude API (Marketing-Chat)**:
   - Paste **API-Schlüssel** (stored in `wp_options` as `anthropic_api_key`).
   - Optional **Modell-ID** (default: `claude-sonnet-4-6`), stored as `anthropic_model`. Older IDs (e.g. Claude 3.5 snapshots) may return 404 after Anthropic deprecations — see [model overview](https://docs.anthropic.com/en/docs/about-claude/models/overview).
3. Open **`/marketing`** while logged in — chat uses agent **`marketing`** (YouTube/marketing system prompt + compact YouTube sync context + **Anthropic tool use / Pattern C**: the model can call server-side tools that query `dls_youtube_*` / analytics chunks; the server runs the tool loop and returns the final assistant reply).

## Agents

Defined in `inc/services/ai-agent-registry.php`:

| Slug        | Use case                          |
|------------|-----------------------------------|
| `marketing`| Senior YouTube/marketing strategist — extended grounding: benchmark bands (CTR, retention, subs, trends), 5-step analysis framework, tone rules, strict no-hallucinated numbers; plus auto-injected short dashboard context; **YouTube tools** (`inc/services/ai-youtube-chat-tools.php`) for real DB-backed metrics |
| `general`  | Generic assistant (for reuse)   |

To mount the chat elsewhere: `import AiChatPanel from "./ai-chat-panel.js"` and `<AiChatPanel agentKey="general" title="…" />`.

## REST (`dls/v1`, logged-in)

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/ai/agents` | List agents |
| GET | `/ai/config` | `has_api_key`, `model` (no secret) |
| POST | `/ai/config` | `api_key`, `model` |
| POST | `/ai/chat/message` | `{ agent, message, session_id? }` |
| GET | `/ai/chat/sessions?agent=marketing` | Session list |
| GET | `/ai/chat/messages?session_id=` | Messages |
| DELETE | `/ai/chat/session/{id}` | Delete session + messages |

## Database

- `{$prefix}dls_ai_chat_session` — per user + `agent_key`
- `{$prefix}dls_ai_chat_message` — `role` user|assistant, `content`

Install: `install-ai-chat-tables.php` on `init` (option `dls_ai_chat_db_version`).

## Costs

Billing is **Anthropic API** usage (tokens), not Claude Pro (claude.ai). See Anthropic pricing.
