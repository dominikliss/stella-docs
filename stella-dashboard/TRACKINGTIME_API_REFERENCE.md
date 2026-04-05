# TrackingTime API v4 — Project reference

> **Saved for:** `ddashboard` theme integration (PHP services, JS `tracking-time-pro-service.js`).  
> **Official:** [developers.trackingtime.co](https://developers.trackingtime.co/)  
> **Base URL (documented):** `https://api.trackingtime.co/api/v4`  
> All requests must use **HTTPS**.

**Full original export (endpoints + examples + release notes):** [`TRACKINGTIME_API_VERBATIM_EXPORT.md`](./TRACKINGTIME_API_VERBATIM_EXPORT.md) (~277k characters). This file is a shorter structured summary.

## Alignment with this theme

| Topic | Official docs | This theme |
|--------|-----------------|--------------|
| Host | `api.trackingtime.co` | PHP proxy uses **`api.trackingtime.co`** (`inc/routes/tracking-time.php`) |
| Browser | — | JS calls **`/dls/v1/tracking-time/*`** via `@wordpress/api-fetch` — **no credentials in the browser** |
| Paths | `GET` + query; optional `/:account_id/...` | Same; account id from option **`tracking_time_pro_account_id`** (0 = default workspace) |
| `User-Agent` | Required | Set server-side: `ddashboard (site url)` |
| Auth | HTTP Basic + App Password | Stored in WP options; **not** passed through `stella_options` to JS |
| Response | `response` + `data` | REST responses wrap as `{ data: … }` matching prior `axios` shape |

---

## Making a request

- Prefix: `https://api.trackingtime.co/api/v4`
- **Multiple workspaces:** use  
  `https://api.trackingtime.co/api/v4/:account_id/:endpoint_path`  
  Example: `https://api.trackingtime.co/api/v4/12345/users`  
- If you omit `account_id`, the **default** workspace account is used.
- Obtain **`account_id`** via the **login** endpoint (per official docs).

---

## Authentication

**HTTP Basic Authentication** — email as username; password = account password or **App Password**.

Steps:

1. Build `email:password`
2. Base64-encode the string
3. Header: `Authorization: Basic <encoded>`

Example (conceptual):

```http
GET /api/v4/users HTTP/1.1
Host: api.trackingtime.co
Authorization: Basic <base64(email:password)>
Content-Type: application/json
User-Agent: ddashboard (https://example.com/contact)
```

---

## Identifying your app (`User-Agent`)

Required: include **User-Agent** with application name and a **contact** (website or email), optionally version.

Examples:

- `MyApp, Inc. (http://myapp.com/contact)`
- `MyApp v1.2 (email@yourapp.com)`

---

## Standard JSON response

All endpoints return a wrapper:

```json
{
  "response": {
    "status": 200,
    "version": "4.0",
    "message": "ok",
    "note": null,
    "note_type": null
  },
  "data": {}
}
```

- **version** — API version that handled the request  
- **status** — logical status (callers should validate; errors may use `500` in `response.status` with a message)  
- **message** — server message  
- **note** / **note_type** — optional user-facing note (`INFO` | `WARNING` | `ERROR` | `ALERT`)

On error, `response.status` may be `500` with an explanatory `message`; always check **`response.status`** inside the JSON body.

---

## Customers

| Method | Path | Notes |
|--------|------|--------|
| GET | `/:account_id/customers/:customer_id` | Query: `include_custom_fields` (bool, default false) |
| GET | `/:account_id/customers` | Query: `include_custom_fields` |
| POST | `/:account_id/customers/update/:customer_id` | Params: `name`, `custom_fields` JSON `[{"id":,"value":"value"}]` |
| GET | `/:account_id/customers/delete/:customer_id` | Irreversible |

---

## Projects

| Method | Path | Notes |
|--------|------|--------|
| GET | `/:account_id/projects/:project_id/min` | Query: `include_tasks`, `include_task_lists`, `include_custom_fields`, `include_closed_tasks` |
| GET | `/:account_id/projects` | Query: `filter` = `ALL` \| `ACTIVE` \| `ARCHIVED` \| `FOLLOWING` (default ACTIVE), `include_custom_fields` |
| GET | `/:account_id/projects/:project_id/users` | Users assigned to project (via tasks) |
| POST | `/:account_id/projects/update/:project_id` | Params include `name`, `is_billable`, `hourly_rate`, `fixed_rate`, `is_public`, `default_view`, `custom_fields`, `customer_id`, `customer_name`, `estimated_time`, … |
| GET | `/:account_id/projects/close/:project_id` | Archive project |
| GET | `/:account_id/projects/open/:project_id` | Re-activate project |

---

## Tasks

| Method | Path | Notes |
|--------|------|--------|
| GET | `/:account_id/tasks/:task_id` | Query: `include_custom_fields` |
| GET | `/tasks/paginated` | Query: `filter` (`ALL` \| `ACTIVE` \| `ARCHIVED` \| `TRACKING`), `page`, `page_size`, `include_custom_fields` |
| GET | `/:account_id/tasks/delete/:task_id` | Query: `delete_all` (default false); deletes related events — dangerous |

---

## Events

Events are time ranges on tasks/projects; list endpoints may use **abbreviated** field names for performance.

### List Events

`GET /:account_id/events` (examples also show `.../events/min`)

Query parameters (typical):

| Param | Description |
|--------|-------------|
| `filter` | `USER` \| `PROJECT` \| `CUSTOMER` \| `TASK` \| `COMPANY` (default `USER`) |
| `id` | Entity id when filter ≠ `COMPANY` |
| `from` | Start date (required) |
| `to` | End date (required) |
| `page`, `page_size` | Pagination (defaults vary; `page_size` max e.g. 20,000 on list events) |
| `order` | `asc` \| `desc` by start date |
| `include_custom_fields` | bool |
| `sort_by` | Optional: `created_at` \| `updated_at` — changes how `from`/`to` apply |

### Abbreviations (List Events “min” style)

| Abbr | Meaning |
|------|---------|
| `a` | archived |
| `b` | billable |
| `bd` | billed |
| `c` | customer name |
| `cid` | customer id |
| `d` | duration |
| `id` | event id |
| `p` | project name |
| `pid` | project id |
| `t` | task name |
| `tid` | task id |
| `s` / `e` | start / end |
| `u` | user name |
| `uid` | user id |
| … | (see original export for full list) |

### List Flat Events

`GET /:account_id/events/flat` — flattened custom field columns on each row.  
`page_size` max e.g. 5,000.

### List Filtered Events

`GET /:account_id/events/filtered` — `query_data` JSON with `date_range`, `parameters`, `filters`, `sorts`, `groups`, `rounding`.  
Custom fields: `cf_{type}_{object_id}` e.g. `cf_event_1234567`.

---

## Users

| Method | Path | Notes |
|--------|------|--------|
| GET | `/:account_id/users/:user_id` | Query: `include_custom_fields` |
| GET | `/:account_id/users/` | Query: `filter` (`ALL` \| `ACTIVE` \| `ARCHIVED`), `include_billing`, `include_custom_fields` |
| GET | `/:account_id/users/:user_id/assigned_tasks` | `filter`, pagination, `sort_by`, `order`, `include_billing` |
| GET | `/:account_id/users/:user_id/projects` | |
| GET | `/:account_id/users/:user_id/trackables` | Query: `project_id`, `only_favorites`, `include_tasks` (default true) |

---

## Custom fields & enums

| Method | Path | Notes |
|--------|------|--------|
| GET | `/:account_id/custom_fields/:custom_field_id/enum_options` | `enabled` filter |
| POST | `/:account_id/enum_options/:enum_option_id/update` | `name` |
| GET | `/:account_id/enum_options/:enum_option_id/delete` | |
| POST | `/:account_id/custom_fields/add` | `name`, `value_type`, `filter_object_class`, optional `color`, `notes`, `cf_index` |
| POST | `/:account_id/custom_fields/:custom_field_id/update` | |
| GET | `/:account_id/custom_fields/:custom_field_id/delete` | |

`value_type`: `boolean` \| `currency` \| `date` \| `enum` \| `number` \| `text` \| `checkbox`  
`filter_object_class`: `event` \| `task` \| `project` \| `user` \| `customer`

---

## Related files in this repo

- [`TRACKINGTIME_API_VERBATIM_EXPORT.md`](./TRACKINGTIME_API_VERBATIM_EXPORT.md) — **full** Postman-style export (includes long API changelog)  
- `inc/routes/tracking-time.php` — server proxy (`dls/v1/tracking-time/*`)  
- `src/services/tracking-time-pro-service.js` — thin client → REST only  

---

## Disclaimer

This file is a **structured project reference** derived from the TrackingTime **API v4** documentation export (Postman-style). For authoritative behavior, use [developers.trackingtime.co](https://developers.trackingtime.co/) and your account’s live responses.
