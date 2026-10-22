# Atlas projects and Trello push

Imported TrackingTime work is shown at **`/projects`**. Project name, notes, customer, archived flag, and **max hours per week** can be edited in the project header (**Edit**). Those edits are **Atlas-local only** — they are stored on the `projects` row and are never written back to TrackingTime. Later import / hourly open-task sync still refreshes tasks and `estimated_seconds`, but does not overwrite those local fields.

When max hours per week is set, the project list and detail show **one** this-week slider against the cap: **commented** hours (trimmed `time_entries.notes` nonempty) and **no comment** as stacked segments. The weekly limit, weekly-hours alerts, and Trello “hours worked” use **commented** hours only. Both sums include archived tasks and subtasks whose `started_at` falls in the current Monday–Sunday week in `Europe/Vienna`. TrackingTime timers still running (no end time) are not imported. Settings → Notification rules can alert (inbox / Slack / email) when the commented bar crosses a threshold — [`notifications.md`](notifications.md). Each project can optionally push **one way** to a Trello board. Trello never writes back into Atlas.

TrackingTime import / hourly open-task sync: [`tools.md`](tools.md#trackingtime-import-and-sync).  
Client stubs: [`../stella-dashboard/atlas-client-lookup.md`](../stella-dashboard/atlas-client-lookup.md).

---

## What is published

Atlas talks to Trello REST v1 with a **key + token** (Settings → Trello). Not OAuth 2.0. Credentials are encrypted in `app_settings` (`trello_api_key`, `trello_token`); env fallbacks `TRELLO_API_KEY` / `TRELLO_TOKEN`.

Per project, the operator maps:

| Atlas | Trello |
|-------|--------|
| Board | One board |
| Active task list | In progress column + Done column |
| Backlog task list | Backlog column |
| Hours worked field (optional) | Number or text custom field (e.g. **Zeit gearbeitet**) |

A mapping is ready when at least one of the two list pairs is complete (active + both columns, or backlog + backlog column). Individual tasks in those lists can be marked **keep off the board** (`tasks.skip_trello`) in the project Trello panel. That flag is Atlas-local and is not overwritten by TrackingTime import.

---

## Push rules

Identity is **`tasks.trello_card_id` only**. Comments are never pushed.

| Atlas task | Trello |
|------------|--------|
| Open, in the mapped active list, no card yet | Create a card in **In progress** |
| Later archived (`is_archived`) and already has a card | Move that card to **Done** and mark it complete (`dueComplete=true`). Position is not forced to the top. |
| Reopened (archived → open again, e.g. in TrackingTime) and has a card | Move the card back to **In progress** (or **Backlog**) and clear complete (`dueComplete=false`) |
| Open, in the mapped backlog list, no card yet | Create a card in **Backlog** |
| `skip_trello` on a mapped-list task | Skip — stays in Atlas, no create/update. An existing card is left on the board as-is |

Card name: `!` / `!!` / `!!!` from `tasks.priority` (1–3), then the task name.

Sort inside a column: priority high first, then tasks with a due date (earliest first), then undated. Position is `(index + 1) * 65536` (lower `pos` = top).

Description: HTML → Trello markdown, plus estimated hours when set, plus a Subtasks checklist-style block, plus a **History** block: one `Name: note` line per commented time entry on the task and its subtasks, oldest first, case-insensitive duplicates per person dropped, no dates or durations. Notes are cut at 200 characters; only the latest 20 lines are kept (`… N earlier entries` above them). A changed note changes the content hash, so the hourly sync re-pushes. Checklists and attachments (≤ 10 MB) are created once and updated by stored Trello ids.

Updates send **`idBoard` + `idList`** so a remapped board can move existing cards. If the stored card is gone (GET 404), Atlas clears the ids and treats the task as unpublished.

---

## Hours worked

Hours = sum of `time_entries.duration_seconds` on the task **and its subtasks** that have a nonempty text comment (`notes`), ÷ 3600. Uncommented entries stay in Atlas but are not pushed.

| Hours field mapping | Where hours go |
|---------------------|----------------|
| Empty | Card description: `Hours worked: 1.00h` |
| Number or text custom field | That field only (not also in the description) |

Number values are a decimal (`1.5`) or a whole number (`2`, not `2.00`). If Trello rejects a float, Atlas retries the rounded integer.

The field must exist **on the mapped board**. `GET /boards/{id}/customFields` only returns number/text fields. A field on another board will not appear in the dropdown.

---

## When a push runs

`PushProjectToTrelloJob` on the `deployments` queue (`TrelloPushStarter`).

| Trigger | Behaviour |
|---------|-----------|
| Project UI **Push to Trello** | `POST /api/projects/{id}/trello/push` — always queues; 409 if already pending/running, 422 if unmapped or Trello is not configured |
| After a TrackingTime import/sync of that project | `TrelloPushStarter::startIfMapped` — no-op if unmapped, unconfigured, or already running. Also **skips** when the mapped content hash matches `projects.trello_pushed_hash` (last successful push). |

There is **no** dedicated Trello cron. Hourly Trello is a side effect of `projects:sync-tracking-time-open` (Laravel scheduler, every hour). Quiet hours (TrackingTime upserted the same values) do not rewrite Trello cards.

The hash covers board/list/hours mapping, skipped task ids (`tasks.skip_trello`), plus candidate tasks (name, description, due, priority, estimate, archive, hours, subtasks, checklists, attachments, whether a card already exists). Comments are ignored. The hash also includes a push format version (`TrelloPusher::PUSH_FORMAT`); bump it when push behaviour changes (e.g. marking Done cards complete) so the next hourly sync re-pushes existing cards once. **Reset Trello IDs** clears the hash so the next auto-push recreates cards.

Progress (`trello_push_status` / `done` / `total` / `log`) is stored on the project and polled by the project page.

---

## Reset Trello IDs

Project Trello panel → **Reset Trello IDs**. Clears this project’s `trello_card_id`, `trello_checklist_id`, check-item ids, and attachment ids. Other projects are left alone. Mapping is kept. Blocked with **409** while a push is pending/running.

Use this after changing to a different board so the next push creates new cards instead of updating cards that still live on the old board.

---

## API

| Method | Path | Purpose |
|--------|------|---------|
| `GET` | `/api/projects` | List imported projects |
| `GET` | `/api/projects/{id}` | Project detail (lists, tasks, `trello` state) |
| `PATCH` | `/api/projects/{id}` | Update name, notes, color, customer, archived, max hours / week |
| `GET` | `/api/projects/{id}/trello` | Mapping + push progress |
| `PATCH` | `/api/projects/{id}/trello` | Save board / columns / task lists / hours field / `skip_trello_task_ids` |
| `POST` | `/api/projects/{id}/trello/push` | Queue a push |
| `POST` | `/api/projects/{id}/trello/reset` | Clear stored Trello ids for this project |
| `GET` | `/api/trello/boards` | Open boards for the saved token |
| `GET` | `/api/trello/boards/{boardId}/lists` | Open lists |
| `GET` | `/api/trello/boards/{boardId}/custom-fields` | Number/text custom fields |
| `GET` | `/api/settings/trello` | Configured? (never returns the key/token) |
| `PATCH` | `/api/settings/trello` | Save key/token (`••••••••` = keep current) |
| `POST` | `/api/settings/trello/test` | `GET /members/me` |

---

## Code

| Piece | Role |
|-------|------|
| `TrelloClient` | REST v1 (`key` + `token` query params) |
| `TrelloPusher` | Create/update/move cards, hours, checklists, attachments |
| `TrelloPushStarter` | Queue guard + hourly hook (skip when content hash matches) |
| `PushProjectToTrelloJob` | `deployments` queue, 30 min timeout, 1 try |
| `TrelloSettings` | Encrypted credentials |
