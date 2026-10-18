# Atlas client lookup (planned)

**Status: not implemented.** Contract for ddashboard REST that Atlas will call. Do not treat these routes as live until the theme ships them and this page’s status is flipped.

Atlas is taking over **TrackingTime Pro project import** (lists, tasks, subtasks, attachments, time entries) into Atlas MySQL. CRM stays in ddashboard. Atlas only needs a slim client stub: ddashboard id, display name, TrackingTime Pro id.

Atlas will **not** call `POST /dls/v1/pm/import-tracking-time-projects` or `/dls/v1/tracking-time/*` for this importer. Those stay as the existing in-theme PM/TrackingTime paths.

When implementing in the theme: add `inc/routes/atlas-clients.php`, register it with the other `dls/v1` routes, then add the three paths to [`reference/dls-v1.openapi.json`](reference/dls-v1.openapi.json).

Related: [`CAPABILITIES.md`](CAPABILITIES.md) (clients + PM), [`architecture.md`](architecture.md), [`../atlas/tools.md`](../atlas/tools.md), [`../atlas/projects.md`](../atlas/projects.md), [`../integration/system-overview.md`](../integration/system-overview.md).

---

## Do not change

- Client CRM (`dls_client`, contacts, invoices, mail)
- `tracking_time_pro_id` storage on the client
- Cookie UI: `GET /dls/v1/clients`, `/wp/v2/client`
- TrackingTime proxy (`/dls/v1/tracking-time/*`) and PM CRUD (`/dls/v1/pm/*`)
- TrackingTime API key in Verwaltung — Atlas will store its own App Password

---

## Allowed IPs (hard allowlist)

These routes must be reachable **only** from Atlas and Stella. Browser traffic, the managed-host portal IPs, and everyone else get **403**, even with a valid Bearer.

| Host | IPv4 |
|------|------|
| Atlas production CX33 | `2.31.42.13` |
| Stella (also hosts `atlas.dev`) | `95.217.144.93` |

Do **not** allow `194.126.177.181` or `23.88.90.12` on these routes.

Resolve the client IP the same way other IP-restricted routes do (`inc/routes/authenticate.php` / `DLS_ALLOWED_IPS` patterns), including the reverse-proxy header the theme already trusts.

Put the two IPs in a named constant (e.g. `DLS_ATLAS_ALLOWED_IPS`) in `inc/constants.php`.

Check **IP first**, then Bearer, then role.

Open gap: Atlas (`2.31.42.13`) currently gets **401** from ddashboard on other paths; Stella (`95.217.144.93`) gets **403** from Apache. These lookup routes still need the two IPs allowed at the application layer. Whether `.htaccess` must also allow them for `/wp-json/dls/v1/atlas/*` is a separate host change — see [`../open-gaps.md`](../open-gaps.md).

---

## Auth

Atlas already uses:

| Method | Path |
|--------|------|
| `POST` | `/dls/v1/auth/login` |
| `POST` | `/dls/v1/auth/refresh` |
| `POST` | `/dls/v1/auth/logout` |
| `GET` | `/dls/v1/auth/verify` |

Every Atlas lookup route must accept the same header:

```http
Authorization: Bearer {access_token}
```

`permission_callback`: same token resolution as `GET /dls/v1/auth/verify`. Allow WP roles `administrator` and `team_member` only. Reject `customer` with **403**. Cookie + `X-WP-Nonce` is not required and must not be the only way in.

Do **not** apply Präsentationsmodus masking. Atlas needs real names and real TrackingTime ids.

---

## Endpoints

Namespace `dls/v1`. Base `{ATLAS_DDASHBOARD_URL}` (no trailing slash), e.g. `https://data3.dominikliss.com/wp-json`.

### `GET /dls/v1/atlas/clients`

List clients for the Atlas TrackingTime importer.

**Query**

| Param | Type | Default | Meaning |
|-------|------|---------|---------|
| `has_tracking_time_pro_id` | `0` or `1` | `1` | `1` = only clients with a non-empty TrackingTime Pro id |
| `status` | `active` \| `lead` \| `inactive` | omitted | optional filter |

No request body.

**200**

```json
{
  "data": [
    {
      "id": 372,
      "name": "Acme GmbH",
      "tracking_time_pro_id": 372222,
      "status": "active"
    }
  ]
}
```

| Field | Type | Source |
|-------|------|--------|
| `id` | integer | `dls_client.id` |
| `name` | string | `official_name`; if empty, client title |
| `tracking_time_pro_id` | integer or `null` | client `tracking_time_pro_id` |
| `status` | string | `active` / `lead` / `inactive` |

When `has_tracking_time_pro_id=1`, every item **must** have a non-null integer `tracking_time_pro_id`. Sort by `name` ASC. No pagination unless the mapped-client count becomes large.

Do not return address, VAT, emails, logo, persons, or invoices.

### `GET /dls/v1/atlas/clients/{id}`

Same object for one client. Return it even if `tracking_time_pro_id` is `null`. **404** if the id does not exist.

**200**

```json
{
  "data": {
    "id": 372,
    "name": "Acme GmbH",
    "tracking_time_pro_id": 372222,
    "status": "active"
  }
}
```

### `GET /dls/v1/atlas/users`

Atlas maps TrackingTime people to WP users by email. Return team users only (`administrator`, `team_member`), never `customer`.

**200**

```json
{
  "data": [
    {
      "id": 1,
      "email": "dominik@foxcraft.digital",
      "name": "Dominik Liss",
      "roles": ["administrator"]
    },
    {
      "id": 2,
      "email": "pawel@foxcraft.digital",
      "name": "Paweł Lubański",
      "roles": ["team_member"]
    }
  ]
}
```

`id`, `email`, `name`, `roles` must match `/dls/v1/auth/verify` and the `user` object on `/auth/login`.

---

## Errors (WordPress REST)

**403 — IP not Atlas/Stella**

```json
{
  "code": "rest_forbidden",
  "message": "Access denied.",
  "data": { "status": 403 }
}
```

**401 — missing/invalid Bearer** (same as `/auth/verify`)

```json
{
  "code": "rest_not_logged_in",
  "message": "You are not currently logged in.",
  "data": { "status": 401 }
}
```

**403 — wrong role** (e.g. `customer`)

```json
{
  "code": "rest_forbidden",
  "message": "Sorry, you are not allowed to do that.",
  "data": { "status": 403 }
}
```

**404 — unknown client id**

```json
{
  "code": "rest_not_found",
  "message": "Client not found.",
  "data": { "status": 404 }
}
```

---

## What Atlas will call (after this exists)

```
Atlas (Bearer from login)
  GET /dls/v1/atlas/clients?has_tracking_time_pro_id=1
    → Tools → TrackingTime Import customer picker

  GET /dls/v1/atlas/clients/{id}
    → upsert Atlas customer stub
       (ddashboard_id, tracking_time_pro_id, name)

  GET /dls/v1/atlas/users
    → seed/map Dominik + Paweł by email

  then TrackingTime API v4 directly
    → projects, lists, tasks, subtasks, files, time entries
```

---

## Theme acceptance

1. Request from a non-allowlisted IP → 403, even with a valid team Bearer.
2. Request from an allowlisted IP without Bearer → 401.
3. Request from an allowlisted IP with a `customer` Bearer → 403.
4. Request from an allowlisted IP with an `administrator` / `team_member` Bearer → 200, slim payload only.
5. `GET /dls/v1/atlas/clients?has_tracking_time_pro_id=1` returns only clients that have a TrackingTime Pro id.
6. Präsentationsmodus does not rewrite names or ids on these routes.
7. Existing `/dls/v1/clients` and cookie UI are unchanged.
