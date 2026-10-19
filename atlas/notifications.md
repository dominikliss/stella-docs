# Atlas notifications

Rules live in **Settings → Notification rules**. Slack incoming webhooks live in **Settings → Slack**. The header bell opens an in-app inbox at **`/notifications`**. Per-project mute / Slack override is on the project page.

Rule types:

| Type | When it fires |
|------|----------------|
| **Weekly hours monitoring** (`project.week_hours_threshold`) | A project’s this-week **commented** hours cross a threshold of its **max hours per week**. Once per rule + project + ISO week + threshold. Week window is Monday–Sunday in `Europe/Vienna` — same as the project progress bar ([`projects.md`](projects.md)). |
| **Time entries with no comment** (`time_entry.missing_comment`) | Selected projects have uncommented time entries (`notes` empty / whitespace) starting **2026-10-19**. Once per rule per Vienna calendar day. Optional `user_ids` (empty = every user). Body lists tasks plus TrackingTime links (`https://pro.trackingtime.co/#/project/{projectId}/{taskId}`). |

---

## How a fire happens

Hourly Laravel scheduler: `notifications:check-week-hours` (`CheckProjectWeekHoursCommand`). Daily 07:00 `Europe/Vienna`: `notifications:check-missing-comments` (`CheckTimeEntryMissingCommentsCommand`). Manual: `php artisan notifications:check-week-hours` (`--project=` limits to one id) or `php artisan notifications:check-missing-comments`. Settings **Test** on a rule calls `NotificationDispatcher::simulate` (does not use real hours).

Weekly hours also requires: project in the rule’s list, not archived, `max_hours_per_week` > 0, binding not muted, this-week **commented** seconds ≥ threshold % of the cap.

Missing-comment requires: project list nonempty, at least one uncommented entry since 2026-10-19 for the selected users (or all users), project not archived.

Dedupe keys:

| Type | Key |
|------|-----|
| Weekly hours | `project.week_hours_threshold:{ruleId}:{projectId}:{isoWeekYear}-W{isoWeek}:{threshold}` |
| Missing comment | `time_entry.missing_comment:{ruleId}:{Y-m-d}` (Vienna date) |

Title / body come from optional rule templates (`{project}`, `{customer}`, `{customer_line}`, `{used}`, `{cap}`, `{threshold}`, `{used_percent}`). Empty templates use the type defaults.

---

## Channels

`NotificationDispatcher` writes one `notification_events` row, then deliveries:

| Channel | When | What |
|---------|------|------|
| In-app | Rule `notify_in_app` | One `notification_deliveries` row per Atlas `users` row. Header unread count. Mark read / mark all read. Click opens `event.url` (`/projects?project={id}`). |
| Slack | Enabled Slack channels on the rule | Incoming webhook `POST` with `text` (title + body). Includes the project URL when set; omits `/notifications`. Project binding `slack_destination_id` **replaces** the rule’s Slack list for that project. |
| Email | Enabled email channel on the rule | Brevo, same as WP-down alerts. Recipient is the channel’s `email_to`, else Settings → Email alert recipient. Skipped if neither is set or Brevo is unconfigured. Same URL rule as Slack. |

Webhook URLs must be `https://hooks.slack.com/…`. Stored encrypted on `slack_destinations`; list/update APIs return `has_webhook`, never the URL (`••••••••` = keep current). Delete is **409** while a rule channel still points at it.

---

## API

| Method | Path | Purpose |
|--------|------|---------|
| `GET` | `/api/settings/slack-destinations` | List destinations (no webhook URL) |
| `POST` | `/api/settings/slack-destinations` | Create (name + HTTPS Slack webhook) |
| `PATCH` | `/api/settings/slack-destinations/{id}` | Rename / replace webhook |
| `DELETE` | `/api/settings/slack-destinations/{id}` | Delete; 409 if a rule still uses it |
| `POST` | `/api/settings/slack-destinations/{id}/test` | Send a test webhook |
| `GET` | `/api/notification-rule-types` | Catalog (labels, placeholders, conditions) |
| `GET` | `/api/notification-users` | Atlas users for the missing-comment rule picker |
| `GET` | `/api/notification-rules` | List rules + channels + project ids |
| `POST` | `/api/notification-rules` | Create |
| `PATCH` | `/api/notification-rules/{id}` | Update |
| `POST` | `/api/notification-rules/{id}/test` | Simulate a fire |
| `DELETE` | `/api/notification-rules/{id}` | Delete |
| `GET` | `/api/notification-events` | Recent fires (Settings log) |
| `GET` | `/api/projects/{id}/notification-bindings` | Mute / Slack override for that project |
| `PATCH` | `/api/projects/{id}/notification-bindings` | Save mute / Slack override |
| `GET` | `/api/notifications` | Current user’s in-app inbox |
| `GET` | `/api/notifications/unread-count` | Unread in-app count |
| `POST` | `/api/notifications/read-all` | Mark all read |
| `POST` | `/api/notifications/{delivery}/read` | Mark one read |

---

## Code

| Piece | Role |
|-------|------|
| `NotificationRule` / `NotificationRuleChannel` | Type, templates, config (`thresholds` or `user_ids`), in-app flag, Slack/email channels |
| `notification_rule_projects` | Which projects a rule watches |
| `NotificationBinding` | Per-project mute + Slack override |
| `NotificationEvent` / `NotificationDelivery` | Fire + per-channel outcome (`sent` / `failed` / `skipped`) |
| `SlackDestination` / `SlackClient` | Encrypted webhook; `POST { text }` |
| `NotificationDispatcher` | Insert event (unique `dedupe_key`), then in-app / Slack / email |
| `ProjectWeekHoursThresholdChecker` | Hourly evaluation + placeholders (commented hours only) |
| `CheckProjectWeekHoursCommand` | `notifications:check-week-hours` |
| `TimeEntryMissingCommentChecker` | Daily 07:00 Vienna evaluation + task list |
| `CheckTimeEntryMissingCommentsCommand` | `notifications:check-missing-comments` |
