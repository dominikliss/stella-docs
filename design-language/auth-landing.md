# Atlas auth landing (2026-11-07)

Snapshot of the unauthenticated screens after the 2026-11-07 design sessions, plus the WCAG 2.1 AA pass that followed. **Not** the shared Technical Precision chrome. The signed-in app still follows [SPEC.md](SPEC.md). Dominik intends a later app-wide redesign; this file is the record of the **new landing language** (pink chrome, glow, 8–12px radius, soft focus) and the AA rules that have to travel with it.

---

## New language at a glance

Copy these when the rest of Atlas is redesigned. Do not copy Technical Precision’s white primary button, 4px radius, or “no glow / no gradient” rules onto a screen that should feel like login.

| Rule | Landing | Technical Precision (signed-in) |
|------|---------|----------------------------------|
| Accent | Atlas pink `#f472b6` **is chrome** (CTA, step, focus, checked) | Pink is identity-only; primary action is white |
| Atmosphere | `layer-0` + pink radials + fading pink grid + faint mark | Flat `layer-0`, no gradients |
| Card | Opaque `layer-1`, **12px** radius, pink border + pink glow | 4px radius, `tech-border`, no shadow |
| CTA | 40px, **8px** radius, `#09090B` on `#f472b6`; hover/active `#ec4899` + pink glow; no white flash | 32–36px, 4px, white fill, no glow |
| Field rest | 40px, **8px** radius, fill `#09090B`, border `outline` `#8e9192` | 4px, `outline-variant` / `tech-border` |
| Field focus | **1px `#f472b6` border** + **3px halo** `rgba(244, 114, 182, 0.2)`. Not a second solid ring (that read as a fat outline). | 1px white border, no glow |
| CTA / checkbox focus | Dark 2px gap + outer 2px white ring (pink-on-black cannot hit 3:1 both ways) | 1px white border |
| Title | 22px / 600 on the card | `h1` 18px |
| Links | Underlined `on-surface-variant`, hover pink | Colour / placement only is not enough next to body text |
| Copy | No slogans. Header wordmark, version, `© foxcraft.digital` | Ops density |

WCAG that is part of this language, not optional polish: rest borders `#8e9192`, field focus = 1px pink (halo is decoration), form errors as `role="alert"` + `aria-invalid`, page title per step, the page must scroll (no `overflow: hidden` trap), underlined links, `prefers-reduced-motion` on the spinner.

**Routes:** `/login`, `/forgot-password`, `/reset-password`  
**Code:** `resources/js/domains/auth/AuthShell.tsx`, `LoginPage.tsx`, `OtpInput.tsx`, `ForgotPasswordPage.tsx`, `ResetPasswordPage.tsx`, `resources/css/app.css` (`.auth-*`)  
**Tests:** `resources/js/domains/auth/auth-a11y.test.tsx`  
**Auth API:** [`../atlas/auth.md`](../atlas/auth.md)

---

## Why it exists

Technical Precision is dense, monochrome, no gradients, no shadows, white primary buttons. The login screen is the one public-facing surface of Atlas, so 2026-11-07 treated it as a **branded landing**, not an ops table.

Inspiration was a dark split layout with a pink grid. The first Atlas pass put sales copy in a left sidebar on that grid; it felt empty. The session then:

1. Kept the pink atmosphere and grid.
2. Dropped the sidebar and sales copy (internal tool — no slogans).
3. Centered a single form card.
4. Kept only `foxcraft.digital`, a `©`, and the Atlas version.
5. Restyled the form to match the pink landing (opaque card, pink CTA, pink focus).
6. Split 2FA into one box per digit.
7. Audited and fixed WCAG 2.1 AA.

Do **not** copy this look into sidebar, tables, Settings, or other signed-in chrome unless a later redesign says so.

---

## Layout

Document flow, not a clipped overlay:

```text
[ header: Atlas mark + “atlas” ]
[ main: centered card, max 420px ]
[ footer: version, then © foxcraft.digital ]
```

- Root: `min-h-dvh`, `overflow-x-hidden`, `layer-0`. Grows with the card so 200% zoom / 320px width / OSK can **scroll vertically** (WCAG 1.4.10). Do not go back to `overflow-hidden` + `main { position:absolute; inset:0 }`.
- Atmosphere, grid, and a faint watermark mark sit in a **fixed** `inset-0 overflow-hidden` layer (`aria-hidden`) so they never add scroll.
- A pink radial glow is absolutely centered behind the card only (`640×480`, `-z-10`).
- Header top-left, footer bottom-left. On a tall screen `main` is `flex-1` and centers the card. On a short screen the column grows and the page scrolls, header and footer included.

Rejected along the way:

- Split sidebar + form (inspiration layout) — sidebar had nothing useful to say.
- Glass / transparent card — looked wrong on the grid; card is opaque `layer-1` (`#0C0C0E`).
- Marketing lines on the left panel.

---

## Colour (auth only)

Uses Technical Precision layers, then **Atlas identity pink as chrome**. That breaks SPEC §2.5 on purpose.

| Role | Value | Notes |
|------|--------|--------|
| Page | `layer-0` `#09090B` | Same as app frame |
| Card | `layer-1` `#0C0C0E` | Opaque |
| Input fill | `layer-0` `#09090B` | |
| Card border | `rgba(244, 114, 182, 0.22)` | Pink, not `tech-border` |
| Card glow | pink / rose box-shadow + radial behind the card | Decorative; `aria-hidden` |
| Atmosphere | radials of `#f472b6` @ 32% (bottom-left) and `#9d174d` @ 38% (top-right) | |
| Grid | 56px pitch, `#f472b6` @ 8%, masked to a vignette | CSS class `.auth-grid` still has 48px — live shell uses 56px inline |
| Step / hover links | `atlas` `#f472b6` | |
| CTA fill | `#f472b6` → hover/active `#ec4899` | Text `#09090B`, weight 600 |
| Input / checkbox / OTP rest border | `outline` `#8e9192` | WCAG 1.4.11; not `tech-border-bright` `#3F3F46` (~2:1) |
| Input / OTP focus | 1px `#f472b6` border + 3px halo `rgba(244, 114, 182, 0.2)` | The 1px border is the 1.4.11 indicator (~7:1). Do not add a second solid ring. |
| CTA / checkbox focus | `0 0 0 2px #0C0C0E, 0 0 0 4px #ffffff` | Double ring: white is adjacent to the dark gap, not to the pink fill |
| Error | `error` `#ffb4ab` | Text + border |
| Text | `primary` / `on-surface` / `on-surface-variant` | Same as SPEC §2.2 |

CTA hover is a **deeper pink + soft pink glow**, not a white flash (that flash broke the button). Active stays `#ec4899` so 13px text stays ≥ 4.5:1.

Checked checkbox / switch-like box: fill `#f472b6`, check `#09090B`.

---

## Type on these screens

SPEC scale, with two auth exceptions.

| Element | Token / size | Notes |
|---------|----------------|-------|
| Wordmark “atlas” | `h1` 18px / 600 | Header |
| Step `01 / 02 · Credentials` | `label-xs` 12px, uppercase, tracking `0.18em`, `text-atlas` | |
| Card title | **22px / 600 / leading 28px** | Larger than `h1` 18px — landing only |
| Description, Remember me, links, CTA label | `body-sm` 13px | |
| Field labels, errors, version | `label-xs` / `mono-sm` 12px | |
| Field values | `body-sm` 13px | WCAG has no minimum size; iOS will zoom fields under 16px |
| OTP digits | 18px JetBrains Mono, 44px boxes | Prevents iOS zoom |
| `©` | 11px, 3px gap, nudged 1px down | Optical alignment with `foxcraft.digital` |
| Footer company | `body-sm` 13px `on-surface-variant` | Full opacity (not `/70`) so it survives the pink glow |

WCAG 2.1 AA does **not** require a minimum font size. 12px/13px is allowed if contrast is 4.5:1 (it is, on this card). Large-text exception (3:1) only applies to the 22px semibold title.

---

## Components on the landing

### Card

- Radius **12px** (`rounded-xl`) — SPEC default is 4px; this is landing-only.
- Padding `lg` / `32px` on `sm+`.
- Inner form class `auth-form` so CSS can override shared `Button` / `TextField` without changing signed-in screens.

### Fields

- Height 40px, radius 8px, horizontal padding 12px.
- Visible `<label>` + `htmlFor`.
- `autocomplete="email"` / `current-password` / `new-password` / `one-time-code` (WCAG 1.3.5).
- Native checkbox is visually hidden; 14px custom box. Hit area includes the “Remember me” label.

### CTA

- Height 40px, full width, radius 8px.
- Selectors: `button[type=submit]` and `.auth-cta` (recovery “Continue to Atlas”).
- Loading: spinner `aria-hidden`, `aria-busy`, `motion-reduce:animate-none`. Label becomes “Signing in…”.

### Links

- Underlined muted text (`authTextLinkClass`). Colour-only links failed 1.4.1 next to the description.
- Hover `text-atlas`. Focus: 2px white ring, offset 2px, offset colour `layer-1`.
- Back link: Material `arrow_back` at 16px, `aria-hidden`; underline on the text only.

### OTP

Six 44px mono boxes, one digit each. Group labelled “Code”. Paste fills from the focused index. Arrow keys move. Backspace clears then moves left. `onComplete` submits when 6 digits are in. Recovery-code path is a normal `TextField` plus “Use a recovery code” / “Use authenticator code”.

### 2FA enroll QR

If a manual secret is on screen, the QR is `aria-hidden` (the secret is the text alternative). If there is a QR and no secret: `role="img"` + `aria-label="QR code for authenticator setup. Scan with Proton Pass."`.

### Version

`#app[data-version]` from `config('app.version')` (`APP_VERSION` or latest git tag). Footer, `mono-sm`, above the copyright line.

---

## WCAG 2.1 AA (what failed, what we changed)

Audit was code-based (this conversation, 2026-11-07). Target: **WCAG 2.1 Level AA**.

### Failures that were fixed

| Criterion | Was | Now |
|-----------|-----|-----|
| **1.4.11** Non-text contrast | Input / OTP / checkbox border `#3F3F46` on `#0C0C0E` ≈ 2:1. Fill `#09090B` vs card is ~1:1, so the border *is* the control edge. | Rest border `#8e9192` (design-token `outline`). |
| **1.4.11** Field focus | First AA pass used 1px + **2px solid** `#f472b6` — contrast-legal but visually a thick outline. | **1px `#f472b6` border** (≈7:1, the actual indicator) + **3px 20% halo**. Do not bring the solid extra ring back. Error focus uses the same halo in `#ffb4ab`. |
| **1.4.11** CTA focus | `box-shadow: 0 0 0 3px rgba(244,114,182,0.35)` ≈ 2:1 vs the page. A solid pink or white ring cannot hit 3:1 against **both** `#f472b6` and `#09090B`. | Dark 2px gap + outer 2px white ring (GDS-style offset indicator). Same on the checkbox. |
| **1.4.3** CTA `:active` | `#09090B` on `#db2777` ≈ 4.4:1 at 13px. | Active = hover `#ec4899` (~5.8:1). |
| **1.4.3** Footer version | `on-surface-variant/70` on the bottom-left pink glow could drop to ~3:1. | Full `on-surface-variant`. |
| **1.3.1 / 4.1.2 / 4.1.3** Errors | Extra `<p>` under the password field only. No `aria-invalid`, no `aria-describedby`, no live region. | `AuthError` `role="alert"` `id="auth-error"`. Both credential fields `aria-invalid` + `aria-describedby`. `TextField` / `OtpInput` do the same when they own the message. Forgot/reset success: `role="status"`. |
| **2.4.2** Page titled | Blade `<title>Atlas</title>` never changed. | `AuthShell` sets `document.title` to `{title} · Atlas`. |
| **1.4.10 / 1.4.4** Reflow | `overflow-hidden` + absolutely centered main **clipped** the card at 200% / 320px / 2FA+QR. | In-flow flex column; vertical scroll allowed. |
| **1.4.1** Use of colour | “Forgot password” was the same 13px muted colour as the description, no underline. | Underline + focus ring. |
| **1.1.1** QR | Inner SVG, no name, secret sometimes absent. | Hidden when secret is present; named `img` otherwise. |
| **2.2.2** Motion | `animate-spin` ignored `prefers-reduced-motion`. | `motion-reduce:animate-none` + global CSS. |

### Already passing (left as-is)

- **1.4.3** text on the card: white, `#c4c7c8`, `#f472b6`, `#ffb4ab` on `#0C0C0E` are all ≥ 4.5:1 (pink on near-black is ~7:1).
- **1.3.5** autocomplete on email / password / OTP.
- **3.3.2 / 2.5.3** visible labels match accessible names.
- **2.1.1** native form, link, submit, checkbox.
- **3.1.1** `html lang`.
- **2.4.6** one `h1` per step (“Sign in”, “Forgot password”, …).
- **2.4.1** skip link not required (no repeated nav).
- **3.3.3** login failures stay generic (“Check your credentials”) on purpose.

### Font size (not an AA fail)

AA has no minimum px size. 12px labels and 13px body are legal here because contrast holds. Practical notes, not WCAG:

- Email/password at 13px will **zoom on iOS** (Safari threshold is 16px). OTP is already 18px.
- Sizes are `px`, not `rem`. Browser **page zoom** still satisfies 1.4.4; “text only” size in the browser will not scale this UI.

### How to keep AA if the landing is redesigned

1. Control edges ≥ 3:1 against the adjacent fill (`outline` `#8e9192` on this dark card works; `#3F3F46` does not).
2. Focus indicator ≥ 3:1 against **both** the control and the background. Fields: 1px pink border is enough; keep the halo soft. CTA: two-colour offset ring (pink-on-black cannot do both with one colour).
3. Associate errors (`aria-invalid`, `aria-describedby`, `role="alert"`). Do not hang a form-level failure only on Password.
4. Change `document.title` per step.
5. Never `overflow: hidden` a centered auth card; the page must be able to scroll at 320px CSS width.
6. Links in body-sized muted text need an underline (or 3:1 vs surrounding text).
7. Keep `prefers-reduced-motion` on looping spinners.

---

## Implementation map

| Piece | Where |
|-------|--------|
| Shell, title, error live region, link classes | `resources/js/domains/auth/AuthShell.tsx` |
| Steps, QR, recovery codes list | `resources/js/domains/auth/LoginPage.tsx` |
| Digit boxes | `resources/js/domains/auth/OtpInput.tsx` |
| Field `aria-*` | `resources/js/components/TextField.tsx` (`invalid`, `role="alert"` on the message) |
| Busy spinner | `resources/js/components/Button.tsx` |
| Pink landing CSS | `resources/css/app.css` `.auth-form`, `.auth-cta`, `.otp-input` |
| Version into the DOM | `resources/views/app.blade.php` `data-version`, `config/app.php` `version` |
| AA regression | `resources/js/domains/auth/auth-a11y.test.tsx` |

Shared `Button` / `TextField` stay SPEC-correct for the signed-in app. Auth overrides are scoped to `.auth-form`.
