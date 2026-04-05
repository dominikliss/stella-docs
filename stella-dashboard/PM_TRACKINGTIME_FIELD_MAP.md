# PM tables ↔ TrackingTime Pro (import phase)

Reserved for a future **server-side import** (`POST` batch under `dls/v1`, not in the browser). Map TrackingTime API entities into `dls_pm_*` using nullable external ID columns.

| PM table / column | TT concept (see `TRACKINGTIME_API_REFERENCE.md`) |
|-------------------|--------------------------------------------------|
| `dls_pm_project.tt_project_id` | TT project id |
| `dls_pm_task_list.tt_task_list_id` | TT list / board id (as applicable) |
| `dls_pm_task.tt_task_id` | TT task id |
| `dls_pm_comment.tt_comment_id` | TT comment id |
| `dls_pm_time_entry.tt_time_entry_id` | TT time entry id |

Import should upsert by `tt_*` id, then set internal FKs (`project_id`, `task_list_id`, `task_id`). Assignees: map TT users to WordPress users where possible, or leave empty for v1.
