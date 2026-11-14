# Atlas authentication

Atlas stores local users and mints its own access / refresh tokens. Login is email + password, then TOTP (or a recovery code). ddashboard is no longer the identity provider. Atlas **is** the identity provider for Edison (authorization-code + PKCE). Edison never receives Atlas API bearers; 2FA stays on Atlas.

System context: [`../integration/system-overview.md`](../integration/system-overview.md).

---

## Why this shape

Ops accounts live in Atlas (`users.password`, TOTP secret, recovery codes, Spatie roles, `is_active`). An admin creates people under **Users**; the first admin comes from `atlas:setup`. ddashboard is still used for **business** APIs (client stubs, TrackingTime ids) with a service token from Settings → ddashboard — not for sign-in.

A leftover `DdashboardTokenDriver` still verifies an unknown bearer against `GET {ATLAS_DDASHBOARD_URL}/dls/v1/auth/verify` so an old ddashboard access token can open the API until it expires. New sessions never go that path.

**SPA chrome** for `/login`, `/forgot-password`, `/reset-password` is the 2026-11-07 **landing language** (pink as chrome, glow, 8–12px radius, 1px pink focus + soft halo), not Technical Precision. Snapshot, comparison table, and WCAG 2.1 AA: [`../design-language/auth-landing.md`](../design-language/auth-landing.md).

---

## Login flow

```
Browser  POST /api/auth/login  { email|username, password, remember? }
   → AuthSessionService::beginLogin
      → local User by email; `kind=agent` is the same generic failure as a wrong password; Hash::check; reject inactive
   → next step (challenge or session)
```

Throttled as `atlas-auth` (`ATLAS_AUTH_MAX_ATTEMPTS` / minute, keyed by email + IP).

| After password | Response |
|----------------|----------|
| `password_must_change` or empty hash | `{ challenge: password, challenge_token }` → `POST /api/auth/login/password` |
| No confirmed TOTP | `{ challenge: two_factor_enroll, challenge_token, otpauth_url, … }` → `POST /api/auth/login/two-factor/confirm` |
| TOTP already on | `{ challenge: two_factor, challenge_token }` → `POST /api/auth/login/two-factor` (app code or recovery code) |
| (should not happen: password + 2FA already done) | session |

A completed challenge calls `AuthTokenService::issue`. The SPA stores both tokens (`resources/js/lib/token-store.ts`) and sends `Authorization: Bearer {access_token}` on later API calls.

Challenges live in cache for `ATLAS_AUTH_CHALLENGE_TTL` (default 600s). Invalid credentials → 401 `Invalid credentials.` (Agents always get this, even with a password — never reveal that the account is an agent.) Deactivated account → 401 `This account is deactivated.`

---

## Tokens

| Piece | Store | Default TTL |
|-------|-------|-------------|
| Access | `auth_access_tokens` (sha256) + Redis `token:{sha256}` | `ATLAS_AUTH_ACCESS_TTL` 900s |
| Refresh | `auth_refresh_tokens` (sha256, family) | `ATLAS_AUTH_REFRESH_TTL` 28800s, or `ATLAS_AUTH_REFRESH_TTL_REMEMBER` 2592000s |

`POST /api/auth/refresh` rotates the refresh token in the same family and issues a new access token. Reusing a revoked refresh token revokes the whole family. `POST /api/auth/logout` revokes the refresh (body) and the current bearer.

---

## Subsequent requests (`auth.token`)

`TokenAuthMiddleware`:

1. **Tests:** if `auth()->check()` (Pest `actingAs`) **and there is no bearer**, skip token lookup. Inactive users still 401. A present `Authorization` header always wins so an OIDC token cannot ride an Atlas session.
2. **Atlas access token:** `AuthTokenService::userFromAccessToken` (hashed row, not expired, user active).
3. **Fallback:** `DdashboardTokenDriver::verify()` → `GET {ATLAS_DDASHBOARD_URL}/dls/v1/auth/verify`. Then `firstOrCreate` local `User` + role sync. A verify timeout returns **401**.

No bearer → 401 `Unauthenticated.`

The SPA (`resources/js/lib/api-client.ts`) only toasts when the JSON body has a `message`. Canceled requests, dropped connections, and gateway **502/504** stay silent so background dashboard polls do not show “An unexpected error occurred.”

`GET /api/auth/me` returns the local user + Spatie role names + `kind` / `ssh_username` / `has_avatar` / `features.stack_api` + `two_factor_confirmed` / `password_must_change` / `is_active` (used to bootstrap the SPA). Photos are not inlined; `has_avatar` true means `GET /api/users/{id}/avatar` will 200 (any signed-in user). Agents never reach this: they cannot log in.

---

## Profile, password, and 2FA

Signed-in user (`/profile`):

| Method | Path | Purpose |
|--------|------|---------|
| `PATCH` | `/api/auth/me` | Name / email (email change needs current password) |
| `POST` | `/api/auth/me/avatar` | JPEG / PNG / WebP, max 2 MB; stored 512px on the local disk (`users.avatar_path`) |
| `DELETE` | `/api/auth/me/avatar` | Remove photo |
| `PUT` | `/api/auth/password` | Change password (min 12, mixed case, number; `uncompromised` in production) |
| `POST` | `/api/auth/two-factor` | Start TOTP enrollment |
| `POST` | `/api/auth/two-factor/confirm` | Confirm TOTP |
| `POST` | `/api/auth/two-factor/recovery` | Rotate recovery codes |

Forgot / reset (IP-filtered, throttled, no token):

| Method | Path | Purpose |
|--------|------|---------|
| `POST` | `/api/auth/forgot-password` | Always 200; emails a one-hour link if the account is an active human. Agents get the same 200 and no mail |
| `POST` | `/api/auth/reset-password` | `{ email, token, password }` |

---

## User admin

Admins only (`auth.admin` → `EnsureAdmin`). UI: `/users`.

`/users` is a card grid (1 / 2 / 3 columns), same `stack-card` language as stacks. Toolbar is search (name, email, or `ssh_username`) + **Add user**. There is no All / Humans / Agents filter — type is only the Human / Agent switch in the add drawer. No page heading or count. Each card: large avatar (initials if none), **Off** if inactive; footer is two columns — left name + email (or the SSH username for an agent), right Agent badge or Admin/Dev + 2FA, then `{ssh_username} · N SSH keys` (or “No SSH username”) when stack-api is on. Click opens the user drawer. Empty: “Add someone with an Atlas login, or an agent.”

| Method | Path | Purpose |
|--------|------|---------|
| `GET` | `/api/users` | List (`kind`, `ssh_username`, `has_avatar`, …). Agent `email` is omitted. |
| `POST` | `/api/users` | Human: temporary password emailed; `password_must_change`. Agent: `{kind: agent, name, ssh_username, keys?, current_password}` — no login, no mail. |
| `PATCH` | `/api/users/{user}` | Name, email, role (`admin` / `developer`), `is_active`, unique `ssh_username`. Agents cannot be given `admin`. |
| `DELETE` | `/api/users/{user}` | After deactivate; also removes the avatar. Linked SSH person is removed only when people writes are allowed (`can_edit`) |
| `GET` | `/api/users/{user}/avatar` | Photo bytes (any signed-in user; 404 if none) |
| `POST` | `/api/users/{user}/avatar` | Same rules as profile |
| `DELETE` | `/api/users/{user}/avatar` | Remove photo |
| `POST` | `/api/users/{user}/reset-password` | New temporary password (humans only) |
| `POST` | `/api/users/{user}/reset-two-factor` | Clear TOTP (humans only) |

`ssh_username` is unique, pattern `^[a-z][a-z0-9_-]{0,31}$`, and cannot be changed once set. SSH keys live on Stella, not in Atlas — [`stacks.md`](stacks.md#user-ssh-keys). Deactivate and delete call `people-remove` for the linked person only when the stack-api flag is on **and** this principal can edit people (`can_edit`). atlas.dev is read-only for people, so those actions only change the Atlas login.

`users.kind` is `human` (default) or `agent` (migration `2026_11_14_221000_add_kind_to_users_table`). Agents have `password` null, no sessions, no password-reset mail, and a placeholder email `agent+<ssh_username>@agents.invalid` (`users.email` stays NOT NULL + unique). `php artisan atlas:agent-create {name} {ssh_username}` creates or links without sending keys.

The last active admin cannot be demoted or deactivated. `atlas:setup` creates the first admin with a local password (and required 2FA on first login).

---

## Role mapping (ddashboard fallback only)

ddashboard WordPress role slugs are mapped only when a leftover ddashboard bearer is verified (`DdashboardTokenDriver::mapRoles`):

| WP role (any of) | Atlas role |
|------------------|------------|
| `administrator` | `admin` |
| `team_member` (and not administrator) | `developer` |
| anything else | `viewer` |

New Atlas users are assigned `admin` or `developer` by an admin. `viewer` is still a seeded Spatie role.

---

## IP allowlist

`AllowedIpMiddleware` reads `config('atlas.allowed_ips')` from `ATLAS_ALLOWED_IPS` (comma-separated).

- Empty list → no IP filter (local/dev only).
- Non-empty and client IP not listed → **403** `Access denied.`
- `GET /api/health` is reachable from non-allowlisted IPs (monitors / Docker healthchecks). The middleware is registered globally in `bootstrap/app.php` via `prepend()`, so `withoutMiddleware()` on the route never worked. `/api/health` now returns early inside `AllowedIpMiddleware` (`$request->is('api/health')`). Fixed and deployed 2026-10-10. **TODO (Dominik):** commit hash / tag.

`routes/api.php` still comments that the health route “Must remain OUTSIDE AllowedIpMiddleware”; the real bypass is the early return in the middleware class, not `withoutMiddleware()`.

Login / refresh / logout / forgot / reset **are** IP-filtered.

Production values: `194.126.177.181,23.88.90.12,95.217.144.93,78.47.218.190` (`78.47.218.190` york, appended 2026-11-07; **order UNVERIFIED**, check with `grep`). **TODO (Dominik):** remove `23.88.90.12` unless something on the managed host really calls Atlas; reason for each entry unverified. See [`server.md`](server.md), [`../york/README.md`](../york/README.md).

---

## Demo mode

`ATLAS_DEMO_MODE=true` binds `FakeTokenAuthDriver` instead of `DdashboardTokenDriver` for the leftover verify fallback. Local password login still uses the `users` table.

---

## Config

```
ATLAS_DDASHBOARD_URL=https://data3.dominikliss.com/wp-json   # no trailing slash; leftover verify + business APIs
ATLAS_DDASHBOARD_SERVICE_TOKEN=   # env fallback; prefer Settings → ddashboard
ATLAS_ALLOWED_IPS=1.2.3.4,5.6.7.8
ATLAS_HTTP_TIMEOUT=30
ATLAS_AUTH_ACCESS_TTL=900
ATLAS_AUTH_REFRESH_TTL=28800
ATLAS_AUTH_REFRESH_TTL_REMEMBER=2592000
ATLAS_AUTH_CHALLENGE_TTL=600
ATLAS_AUTH_MAX_ATTEMPTS=5
ATLAS_OIDC_ISSUER=https://atlas.foxcraft.digital
ATLAS_OIDC_PRIVATE_KEY=base64:…    # php artisan atlas:oidc-key --generate
ATLAS_OIDC_EDISON_SECRET=
ATLAS_OIDC_EDISON_REDIRECT_URIS=https://edison.dev.foxcraft.digital/auth/atlas/callback
ATLAS_DEMO_MODE=false
```

---

## OIDC for Edison

First-party authorization-code + PKCE. Atlas API access / refresh tokens stay on `atlas.foxcraft.digital`. Edison exchanges a one-time code for an **ID token** (`aud=edison`) and keeps its own session. Password and TOTP happen only on Atlas.

Generate keys: `php artisan atlas:oidc-key --generate`. Put `ATLAS_OIDC_PRIVATE_KEY` and `ATLAS_OIDC_EDISON_SECRET` in Atlas `.env`. Edison gets the same secret as its client secret, `client_id=edison`.

| Piece | URL |
|-------|-----|
| Discovery | `GET /.well-known/openid-configuration` |
| JWKS | `GET /oauth/jwks` |
| Authorize (browser / SPA) | `GET /oauth/authorize` → if signed in, `POST /api/oauth/authorize` with the Atlas SPA bearer, then redirect to Edison with `code` |
| Token (Edison server) | `POST /oauth/token` (`grant_type=authorization_code`, PKCE verifier, client secret) |
| Userinfo | `GET /oauth/userinfo` with the **OIDC** access token (`oidc_…`), not an Atlas API bearer |

Authorize query: `client_id`, `redirect_uri` (exact match), `response_type=code`, `scope=openid profile email`, `state`, `nonce`, `code_challenge`, `code_challenge_method=S256`. `prompt=login` forces Atlas password + TOTP even if a session exists.

Token response: `{ id_token, access_token, token_type, expires_in }`. `access_token` is only valid at `/oauth/userinfo`. It must not open `/api/*`. ID token claims: `iss`, `sub` (Atlas user id), `aud=edison`, `email`, `name`, `roles` (`admin` / `developer` / `viewer`), `nonce` if sent.

Edison client (not in this repo):

1. Redirect the browser to Atlas `/oauth/authorize` with PKCE S256.
2. Callback receives `code` (never an Atlas bearer).
3. Server POST `/oauth/token` with `client_secret` + `code_verifier`.
4. Verify `id_token` against JWKS (`iss`, `aud=edison`, `exp`, `nonce`).
5. Create an Edison session. Map `roles` as-is. No Edison TOTP.

IP allowlist still applies. `edison.dev` token exchange comes from Stella (`95.217.144.93`), already on `ATLAS_ALLOWED_IPS`. Edison prod on `178.105.203.54` would need that IP added before it can call `/oauth/token`.

---

## What this is not

- **Not Laravel Sanctum** for the live API. `laravel/sanctum` is in `composer.json` from the starter; live bearers are rows in `auth_access_tokens`.
- **Not** ddashboard login. Client lookup still uses a ddashboard **service** token ([`../stella-dashboard/atlas-client-lookup.md`](../stella-dashboard/atlas-client-lookup.md)).
- **Not** the SSH-signature scheme used against Stella. That key is for deploy-api / health-api only — see [`../integration/system-overview.md`](../integration/system-overview.md#4-shared-ssh-key-the-one-secret-that-spans-three-jobs).
- **Not** a public IdP. Only configured first-party clients (Edison). Not for ddashboard.
