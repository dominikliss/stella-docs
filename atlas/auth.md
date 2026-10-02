# Atlas authentication

Atlas does **not** store passwords or mint its own API tokens. Identity is **ddashboard**. Atlas is a relying party.

System context: [`../integration/system-overview.md`](../integration/system-overview.md).

---

## Why this shape

ddashboard already has WordPress users and the `dls/v1` auth API. Duplicating accounts in Atlas would drift. Atlas keeps a thin local `users` row (email, name, last seen, Spatie roles, team) so the ops UI has something to attach history to (`initiated_by` on IMAP jobs, etc.).

---

## Login flow

```
Browser  POST /api/auth/login  { username, password }
   → Atlas AuthController
      → POST {ATLAS_DDASHBOARD_URL}/dls/v1/auth/login
         ← { access_token, refresh_token, expires_in, user: { id, email, name, roles[] } }
   → firstOrCreate local User (match email)
   → map WP roles → Atlas roles, sync Spatie
   → Cache::put('token:' . sha256(access_token), { user_id, roles }, expires_in)
   ← { access_token, refresh_token, expires_in, token_type, user }
```

The SPA stores both tokens (`resources/js/lib/token-store.ts`) and sends `Authorization: Bearer {access_token}` on every later API call.

| ddashboard status | Atlas response |
|-------------------|----------------|
| 401 | 401 `Invalid credentials.` |
| other non-2xx | 503 `Authentication service unavailable.` |
| 2xx missing `user.id/email/name` | 502 `Unexpected response from authentication service.` |

---

## Refresh and logout

| Atlas | ddashboard | Notes |
|-------|------------|-------|
| `POST /api/auth/refresh` `{ refresh_token }` | `POST /dls/v1/auth/refresh` | New access token cached. Cache value may be `null` (no user id) — next API call then hits `/verify`. |
| `POST /api/auth/logout` `{ refresh_token }` | `POST /dls/v1/auth/logout` | Also `Cache::forget` the current bearer. Always 204. |

---

## Subsequent requests (`auth.token`)

`TokenAuthMiddleware`:

1. **Tests:** if `auth()->check()` (Pest `actingAs`), skip remote verify.
2. **Cache hit:** `token:{sha256(bearer)}` with `user_id` → load `User`, touch `last_seen_at`, bind team.
3. **Cache miss:** `DdashboardTokenDriver::verify()` → `GET {ATLAS_DDASHBOARD_URL}/dls/v1/auth/verify` with the same bearer. Expect `{ id, email, name, roles[] }`. Then `firstOrCreate` + role sync.

No bearer → 401 `Unauthenticated.`

`GET /api/auth/me` returns the local user + Spatie role names (used to bootstrap the SPA).

---

## Role mapping

ddashboard returns WordPress role slugs. Atlas uses its own set (`DdashboardTokenDriver::mapRoles`):

| WP role (any of) | Atlas role |
|------------------|------------|
| `administrator` | `admin` |
| `team_member` (and not administrator) | `developer` |
| anything else | `viewer` |

---

## IP allowlist

`AllowedIpMiddleware` reads `config('atlas.allowed_ips')` from `ATLAS_ALLOWED_IPS` (comma-separated).

- Empty list → no IP filter (local/dev only).
- Non-empty and client IP not listed → **403** `Access denied.`
- `GET /api/health` is registered **without** this middleware so monitors / Docker healthchecks work from non-allowlisted IPs.

Login/refresh/logout **are** IP-filtered.

---

## Demo mode

`ATLAS_DEMO_MODE=true` binds `FakeTokenAuthDriver` instead of `DdashboardTokenDriver`. No call to ddashboard. For UI/dev only.

---

## Config

```
ATLAS_DDASHBOARD_URL=https://<ddashboard-host>   # no trailing slash; Atlas appends /dls/v1/auth/*
ATLAS_ALLOWED_IPS=1.2.3.4,5.6.7.8
ATLAS_HTTP_TIMEOUT=30
ATLAS_DEMO_MODE=false
```

---

## What this is not

- **Not Laravel Sanctum** for the live API. `laravel/sanctum` is in `composer.json` from the starter; the bearer Atlas accepts is a ddashboard access token.
- **Not** a second password store.
- **Not** the SSH-signature scheme used against Stella. That key is for deploy-api / health-api only — see [`../integration/system-overview.md`](../integration/system-overview.md#4-shared-ssh-key-the-one-secret-that-spans-three-jobs).
