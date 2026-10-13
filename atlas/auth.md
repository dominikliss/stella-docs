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
| `POST /api/auth/refresh` `{ refresh_token }` | `POST /dls/v1/auth/refresh` then `GET /dls/v1/auth/verify` | New access token cached as `{ user_id, roles }` with `expires_in`. Verify is best-effort — refresh still succeeds if verify fails. |
| `POST /api/auth/logout` `{ refresh_token }` | `POST /dls/v1/auth/logout` | Also `Cache::forget` the current bearer. Always 204. |

---

## Subsequent requests (`auth.token`)

`TokenAuthMiddleware`:

1. **Tests:** if `auth()->check()` (Pest `actingAs`), skip remote verify.
2. **Cache hit:** `token:{sha256(bearer)}` with `user_id` → load `User`, touch `last_seen_at`, bind team.
3. **Cache miss:** `DdashboardTokenDriver::verify()` → `GET {ATLAS_DDASHBOARD_URL}/dls/v1/auth/verify` with the same bearer. Expect `{ id, email, name, roles[] }`. Then `firstOrCreate` + role sync, and write `{ user_id, roles }` into cache for `ATLAS_TOKEN_CACHE_TTL` (default 3600s). A verify timeout or connection error returns **401**, not 500.

No bearer → 401 `Unauthenticated.`

The SPA (`resources/js/lib/api-client.ts`) only toasts when the JSON body has a `message`. Canceled requests, dropped connections, and gateway **502/504** stay silent so background dashboard polls do not show “An unexpected error occurred.”

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
- `GET /api/health` is reachable from non-allowlisted IPs (monitors / Docker healthchecks). The middleware is registered globally in `bootstrap/app.php` via `prepend()`, so `withoutMiddleware()` on the route never worked. `/api/health` now returns early inside `AllowedIpMiddleware` (`$request->is('api/health')`). Fixed and deployed 2026-10-10. **TODO (Dominik):** commit hash / tag.

`routes/api.php` still comments that the health route “Must remain OUTSIDE AllowedIpMiddleware”; the real bypass is the early return in the middleware class, not `withoutMiddleware()`.

Login/refresh/logout **are** IP-filtered.

Production values: `194.126.177.181`, `23.88.90.12` (managed host), `95.217.144.93` (Stella). **TODO (Dominik):** remove `23.88.90.12` unless something on the managed host really calls Atlas; reason for each entry unverified. See [`server.md`](server.md).

---

## Demo mode

`ATLAS_DEMO_MODE=true` binds `FakeTokenAuthDriver` instead of `DdashboardTokenDriver`. No call to ddashboard. For UI/dev only.

---

## Config

```
ATLAS_DDASHBOARD_URL=https://data3.dominikliss.com/wp-json   # no trailing slash; Atlas appends /dls/v1/auth/*
ATLAS_ALLOWED_IPS=1.2.3.4,5.6.7.8
ATLAS_HTTP_TIMEOUT=30
ATLAS_TOKEN_CACHE_TTL=3600   # used after /verify when expires_in is unknown; login/refresh prefer expires_in
ATLAS_DEMO_MODE=false
```

---

## What this is not

- **Not Laravel Sanctum** for the live API. `laravel/sanctum` is in `composer.json` from the starter; the bearer Atlas accepts is a ddashboard access token.
- **Not** a second password store.
- **Not** the SSH-signature scheme used against Stella. That key is for deploy-api / health-api only — see [`../integration/system-overview.md`](../integration/system-overview.md#4-shared-ssh-key-the-one-secret-that-spans-three-jobs).
