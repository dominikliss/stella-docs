# Edison run options — modes, models, settings

**Status:** Phase 1 (host API) **done 2026-11-02**. Phase 2 (Laravel) and Phase 3 (UI) implemented in the Edison app (2026-11-02). **UNVERIFIED:** end-to-end on `edison.dev`.

This file is the design. [`host-api.md`](host-api.md) holds the live contract. Do not invent values here that are not in that file.

The web app stays agent-agnostic. The host maps a **canonical** `mode` / `model` / `settings` onto Cursor CLI now, Claude Code and OpenCode later. The UI never sends vendor flags.

---

## Decision

1. **Host API first.** Laravel has nothing to forward until `GET /v1/capabilities` and the new `POST /v1/runs` fields exist. Done 2026-11-02.
2. **Additive only.** Existing `POST /v1/runs` bodies stay valid. Omitting `mode` / `model` / `settings` keeps the pre-change argv (Cursor agent + Auto / `EDISON_CURSOR_MODEL`).
3. **Capabilities come from the host.** Laravel does not hardcode mode lists or model IDs. The UI renders what `GET /v1/capabilities` returns for the selected `agent`.
4. **Canonical modes are three:** `agent`, `plan`, `ask`. Cursor IDE extras (Debug, Multitask, Fast, Effort) are **not** first-class fields. If a backend can do them, they appear in `settings` or extra `modes` on capabilities.
5. **Model IDs are opaque strings** from that backend’s catalog. Do not translate ids in Laravel. Example catalog id: `cursor-grok-4.6-high` (not `grok-4.6`).
6. **Ask is not a security boundary.** `--mode=plan|ask` plus `--sandbox enabled` refused a write in the 2026-11-02 sandbox test, but this is not a substitute for a worktree. Guard text still applies.

---

## Phase 1 — host API (done 2026-11-02)

Live at `https://api.edison.foxcraft.digital`. Verified from Edison and from Stella through Caddy. Full contract: [`host-api.md`](host-api.md).

### Closed questions

| Question | Answer (2026-11-02) |
|---|---|
| Does `cursor-agent --mode=ask\|plan` work? | Yes, on CLI **`2026.10.22-27dcf5f`**. `--mode` choices: `plan`, `ask`. `--plan` is shorthand for `--mode=plan`. There is no `--mode=agent`. |
| `cursor-agent models` format? | `<id> - <label>` per line, plus `Available models` header and `Tip:` footer. Parser: `^([A-Za-z0-9][A-Za-z0-9._-]*) - (.+)$`. Live catalog: **270** models, `auto` first. |
| Does `EDISON_CURSOR_MODEL` still override Auto? | Yes. Ops pin wins over Auto. An explicit client model wins over the pin. |

### Sandbox

Plan/ask add `--sandbox enabled` (CLI choices: `enabled`, `disabled`). `EDISON_SANDBOX_READONLY` default `1`; `0`/`false`/`no` drops the flag. Ask + sandbox: prompt asked to create `sandbox-test.txt` — listed folders, refused to write; file did not exist afterwards. Reads on the sshfs mount work with sandbox enabled.

### Catalog size

~270 models. Phase 3 UI needs search or grouping, not a plain dropdown.

---

## Phase 2 — Laravel app (done in repo 2026-11-02)

Do this after the host fields are live (they are). Laravel forwards canonical ids only.

### Driver contract

`AgentDriver` gains options. Keep a single DTO so Fake and Host stay aligned.

```php
final readonly class RunOptions
{
    public function __construct(
        public string $mode = 'agent',
        public ?string $model = null,
        public array $settings = [],
    ) {}
}
```

```php
public function capabilities(?string $agent = null): array;

public function run(Repo $repo, string $prompt, ?string $sessionId = null, ?RunOptions $options = null): AgentResult;

public function stream(Repo $repo, string $prompt, ?string $sessionId = null, ?RunOptions $options = null): Generator;
```

`HostAgentDriver::postRun` adds `mode`, `model`, `settings` to the JSON body when set.

`FakeAgentDriver::capabilities()` returns the same three modes plus `[{ id: auto, default: true }, { id: fake-echo }]`. Fake replies may mention the mode so feature tests can assert forwarding without the host.

### `GET /api/repos` — proxy capabilities

Extend the existing `agent` object. One round-trip for the UI.

```json
{
  "repos": [ "…" ],
  "agent": {
    "driver": "host",
    "configured": true,
    "backend": "cursor",
    "backends": ["cursor", "claude"],
    "health": { "status": "ok", "agents": ["cursor"] },
    "capabilities": {
      "agent": "cursor",
      "modes": [ "…" ],
      "default_mode": "agent",
      "models": [ "…" ],
      "settings": []
    }
  }
}
```

If the host is down or unconfigured: omit `capabilities` or return Fake’s catalog. UI falls back to Agent + Auto, no picker crash.

Cache the host response in Laravel for 60 s (`Cache::remember`). Capabilities are not per-user.

### `POST /api/chats/{id}/messages`

```json
{
  "content": "What starts the ingest pipeline?",
  "mode": "ask",
  "model": "cursor-grok-4.6-high",
  "settings": {}
}
```

| Field | Validation |
|---|---|
| `content` | unchanged: required, string, 1–20000 |
| `mode` | optional, string, max 32. If capabilities loaded: `Rule::in(mode ids)`. If host down: allow `agent`/`plan`/`ask` only |
| `model` | optional, string, max 64 |
| `settings` | optional, array |

Laravel does **not** need to re-validate model IDs against the catalog if the host will 400. Re-validating `mode` against the three canonical IDs is enough for a fast 422.

Stream and JSON paths both accept the same body. `ChatService::send` / `stream` pass `RunOptions` through.

### Persist last choice on the chat

Migration on `chats`:

| Column | Type | Default |
|---|---|---|
| `mode` | string, 32 | `agent` |
| `model` | string, 64, nullable | `null` (means Auto) |
| `settings` | json | `{}` |

Fillable + casts (`settings` → `array`). Include in `ChatController` summary/detail so the composer can restore the last picker state when opening a thread.

On each successful send: write the submitted (or previous) values back. If the request omits `mode`/`model`, reuse the chat’s stored values, then send those to the host. First message on a new chat with no fields → `agent` + Auto.

Do **not** put mode/model on `chat_messages` in v1.

`POST /api/chats` may accept optional `mode` / `model` so a new thread can start in Plan. Not required for v1 if the composer always sends them on the first message.

### Laravel tests

Extend `tests/Feature/AgentChatTest.php` and the host-driver HTTP fakes:

- message without new fields still 201 (regression)
- message with `mode=ask` `model=cursor-grok-4.6-high` → Host fake received those keys
- `mode=nope` → 422
- `GET /api/repos` includes `agent.capabilities.modes`
- Fake driver still works when `EDISON_HOST_API_URL` is empty
- chat row stores last `mode` / `model`

After feature tests pass, run `php artisan test --compact`.

---

## Phase 3 — UI (done in repo 2026-11-02)

- Composer: mode menu (Agent / Plan / Ask) and model menu from `agent.capabilities`
- Model catalog is ~270 entries: **search or grouping**, not a plain `<select>`
- Hide `settings` while the array is empty
- Do **not** add Debug / Multitask / Fast until capabilities lists them
- Disable write-looking chrome when `writes: false` (copy only; still not a lock)
- Persist selection via the chat payload, not only `localStorage`

Copy patterns from `SettingsPage.tsx` / `design/components.html`. No new visual language.

---

## What not to do

| Temptation | Why not |
|---|---|
| Hardcode Cursor IDE modes in React | Claude/OpenCode will not honour Debug/Multitask/Fast |
| Prompt-prefix “you are in ask mode” in Laravel | Does not stop `--force` writes |
| Put `--mode` / `--permission-mode` in the Laravel repo | Breaks the adapter rule in `host-api.md` |
| Share one global model list across agents | IDs differ (`cursor-grok-4.6-high` vs `sonnet` vs `anthropic/…`) |
| Treat `mode=ask` as authorisation | Cursor plan/ask is not a sandbox |
| Version the URL (`/v2/runs`) | Additive fields are enough |
| Store vendor argv on the chat | Host mapping will change; persist canonical ids only |

---

## Implementation order

1. ~~**Host:** `GET /v1/capabilities`.~~ Done 2026-11-02.
2. ~~**Host:** accept `mode` / `model` / `settings` on `POST /v1/runs`.~~ Done. Default path bit-identical to today.
3. ~~**Host:** Cursor argv mapping; drop `--force` for plan/ask; `--sandbox enabled`.~~ Verified Ask + resume-as-agent on osgar.
4. **Docs:** this file stays the design; `host-api.md` holds the live contract.
5. ~~**Laravel:** `RunOptions` + Host/Fake drivers + `GET /api/repos` capabilities + message validation + `chats.mode/model/settings`.~~ Done in the Edison repo 2026-11-02.
6. ~~**UI:** picker bound to capabilities, with search over the ~270-model catalog.~~ Done in the Edison repo 2026-11-02. **UNVERIFIED** on `edison.dev` until deploy.

---

## Claude / OpenCode (later, not built)

Same HTTP body. Suggested mapping when those containers exist:

| Canonical `mode` | Claude argv | OpenCode message field |
|---|---|---|
| `agent` | `--permission-mode auto` (unattended container) | `"agent": "build"` |
| `plan` | `--permission-mode plan` | `"agent": "plan"` |
| `ask` | `--permission-mode plan` (no named ask) | `"agent": "plan"` or a custom `ask` agent |

Until then, `agent: "claude"` → **400** `{ "message": "Agent claude is not installed." }`.
