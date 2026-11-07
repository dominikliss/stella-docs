# Technical Precision — spec

**Name:** Atlas — Technical Precision  
**Version:** 1.0  
**Tokens:** [`tokens.json`](tokens.json)  
**Reference:** Atlas ops app (`https://atlas.foxcraft.digital`)

Use this when building any internal tool that submodules `stella-docs`. The UI recedes; data and operations stay in front.

**Not** [`../stella-dashboard/DESIGN_SYSTEM.md`](../stella-dashboard/DESIGN_SYSTEM.md). That is ddashboard.

---

## Hard rules

- Dark only. No light theme.
- No box-shadows, blurs, or glass in **app chrome**. (Atlas cheats on the mobile nav backdrop. Auth landing is a documented exception — [auth-landing.md](auth-landing.md) — do not copy it into signed-in screens.)
- No decorative gradients in **app chrome**. Auth landing uses pink radials and a grid; that is landing-only.
- Depth = darker/lighter surfaces + 1px borders.
- Information density over whitespace.
- Status colour is for **state only**, never as a brand accent on chrome.
- Atlas pink / Stella sky / Edison green are **product identity** for those three systems, not shared chrome. See §2.5. Auth landing is the exception that uses Atlas pink as chrome.

Canonical values are the **live tokens** (this file + `tokens.json`), not the Zinc prose that still sits in Atlas `design/DESIGN.md`.

---

## 1. Personality

Focused, professional, dense. Developer-tool, not marketing. Control and reliability over decoration. No gradients, no shadows, no rounded blobs.

---

## 2. Colour

### 2.1 Elevation (app chrome)

| Token | Hex | Use |
|-------|-----|-----|
| `layer-0` | `#09090B` | App frame, deepest background |
| `layer-1` | `#0C0C0E` | Sidebar, page header, secondary panels |
| `layer-2` | `#18181B` | Cards, modals, popovers, table row hover |
| `tech-border` | `#27272A` | Default 1px divider / component outline |
| `tech-border-bright` | `#3F3F46` | Floating surfaces (modal, dropdown) |

Do not use MD3 `surface` (`#141313`) as the app frame. Frame is `layer-0`.

### 2.2 Text

| Token | Hex | Use |
|-------|-----|-----|
| `primary` | `#ffffff` | Headings, primary labels, active text, primary button fill |
| `on-surface` | `#e5e2e1` | Body text |
| `on-surface-variant` | `#c4c7c8` | Muted text, captions, inactive nav |
| `on-primary` | `#2f3131` | Text on a white (primary) button |

Do not use the Zinc-400 / Zinc-500 hexes from older Atlas `DESIGN.md` prose.

### 2.3 Surfaces (forms, containers)

| Token | Hex | Use |
|-------|-----|-----|
| `background` / `surface` | `#141313` | Page canvas inside the frame |
| `surface-container-lowest` | `#0e0e0e` | Input fill, lowest inset |
| `surface-container` | `#201f1f` | Active nav item, hover wash |
| `surface-container-high` | `#2a2a2a` | Hover on secondary buttons |
| `surface-container-highest` | `#353434` | Unchecked switch track |
| `outline-variant` | `#444748` | Input / checkbox border at rest |
| `error` | `#ffb4ab` | Error text / input border |

Full MD3 role set (including unused `secondary` / `tertiary` / `fixed` variants) is in `tokens.json`. Prefer the rows above for new UI.

### 2.4 Actions and status

| Token | Hex | Use |
|-------|-----|-----|
| `primary` | `#ffffff` | Primary button fill |
| `primary-fixed` | `#e2e2e2` | Primary hover |
| `status-success` | `#34d399` | Up / ok / healthy |
| `status-success-bg` | `#064e3b` | Success chip tint |
| `status-warning` | `#fbbf24` | Degraded / at-risk |
| `status-danger` | `#f87171` | Error icon / toast |
| `status-danger-text` | `#fca5a5` | Destructive button label |
| `status-danger-border` | `#991b1b` | Destructive button border / hover |
| `status-danger-bg` | `#7f1d1d` | Destructive button fill |
| `status-unknown` | `#6b7280` | No-data / cancelled |

### 2.5 Product identity (optional)

Use only when a screen names Atlas, Stella, or Edison as an entity (monitoring chips, system map). Never as the app accent.

| Token | Hex | Tint |
|-------|-----|------|
| `atlas` | `#f472b6` | `#831843` |
| `stella` | `#7dd3fc` | `#0c4a6e` |
| `edison` | `#86efac` | `#14532d` |

### 2.6 Scrollbars

| Token | Hex |
|-------|-----|
| thumb | `#3F3F46` |
| thumb hover | `#52525B` |

Track is transparent. App chrome 6px; terminal 4px.

---

## 3. Typography

Two families only.

- **Inter** — UI, nav, body
- **JetBrains Mono** — SHAs, IPs, logs, JSON, latency, IDs, field-error helpers

| Role | Family | Size | Weight | Line | Tracking | Use |
|------|--------|------|--------|------|----------|-----|
| `h1` | Inter | 18px | 600 | 24px | -0.02em | Page title, product name |
| `h2` | Inter | 16px | 600 | 24px | -0.01em | Section title, top-bar title |
| `body-base` | Inter | 14px | 400 | 20px | — | Nav, default body |
| `body-sm` | Inter | 13px | 400 | 18px | — | Buttons, table primary cell, form text |
| `label-xs` | Inter | 12px | 500 | 16px | wider | Table headers, sidebar groups, chips |
| `mono-base` | JetBrains Mono | 13px | 400 | 20px | — | Code in content |
| `mono-sm` | JetBrains Mono | 12px | 400 | 16px | — | Logs, numeric columns, chips, field-error helpers |

Rules:

- Base size is 14px. Do not invent a larger marketing scale.
- Sidebar group labels: `label-xs`, uppercase, wide tracking, `on-surface-variant`.
- Keep the 4px vertical rhythm via the line heights above.

---

## 4. Spacing and layout

4px / 8px grid.

| Token | Value | Use |
|-------|-------|-----|
| `xs` | 4px | Tight gaps, chip padding-x |
| `icon-text` | 6px | Icon-to-label inside a button |
| `sm` | 8px | Component padding, nav item padding |
| `gutter` | 12px | Internal panel padding |
| `md` | 16px | Page padding, header padding |
| `lg` | 24px | Modal padding, large section gap |
| `sidebar-width` | 220px | Fixed sidebar |

### Shell

```text
[ sidebar 220px | main (fluid) ]
                 [ top bar 48px ]
                 [ page header   ]
                 [ content       ]
```

- App root: `h-dvh`, `overflow-hidden`, `layer-0`.
- Sidebar: `layer-1`, right border `tech-border`.
- Top bar: 48px (`h-12`), `layer-1`, bottom border `tech-border`.
- Content: scroll inside main, page padding `md` (16px).
- Dashboard grids may use 12 columns. Operational views are flex stacks.

### Breakpoints

| Viewport | Behavior |
|----------|----------|
| `< 768px` | Sidebar becomes a drawer from a menu button. One pane at a time. Sub-nav becomes a horizontal tab strip. Drawers go full width. Tables hide secondary columns or scroll-x. Row actions stay visible (no hover-only). |
| `768–1024px` | Sidebar fixed. Margins 24px. |
| `> 1024px` | Full fluid, or content max `1440px` when the view is not a data table. |

---

## 5. Shape and elevation

- Default radius: **4px** (`0.25rem`) on buttons, inputs, cards, nav items.
- Pills: `9999px` — **status badges only**.
- Terminal / code flush to a panel edge: `0`.
- Every surface: `1px solid tech-border`.
- Floating layer (modal, dropdown): `1px solid tech-border-bright`.
- No shadow. No blur.

---

## 6. Icons

Material Symbols Outlined.

- Variation: `FILL 0`, `wght 300`, `GRAD 0`, `opsz 24`
- Nav / buttons: 18px
- Table rows: 16px
- Top-bar menu: 22px
- Modal warning: 20px

---

## 7. Components

### Buttons

Height is tight: **32–36px**. Font `body-sm` (13px). Radius 4px. Gap icon-to-label `6px`. Disabled: opacity 40–50%, `not-allowed`.

| Kind | Fill | Text | Border | Hover |
|------|------|------|--------|-------|
| Primary | `primary` `#ffffff` | `on-primary` | none | `primary-fixed` `#e2e2e2` |
| Secondary | transparent | `on-surface` | `tech-border` | `surface-container-high` |
| Destructive | `status-danger-bg` | `status-danger-text` | `status-danger-border` | `status-danger-border` |
| Icon-only | transparent | `on-surface-variant` | none | text `primary`, wash `surface-container-high` |

No glow. No shadow. Do not use leftover Breeze buttons (gray/indigo, uppercase tracking).

### Inputs

- Fill: `surface-container-lowest` or `layer-0`
- Border rest: `outline-variant` or `tech-border`
- Text: `body-sm` + `primary` or `on-surface`
- Placeholder: `on-surface-variant` at ~70%
- Focus: **1px `primary` border, no glow, no ring offset**
- Select: custom chevron (no native arrow), right padding 36px
- Technical values (tokens, JSON): `mono-sm` on a near-black fill

### Field errors

Do not invent a second type style (no Inter body, no banner, no icon, no filled error container). Signed-in uses peach `error` `#ffb4ab`. Auth landing uses `status-danger` `#f87171` so the helper does not look like faded Atlas pink.

| Part | Treatment |
|------|-----------|
| Colour | Token `error` `#ffb4ab` on label, 1px control border, and helper. Not `status-danger` (`#f87171`) — that is toast / status-icon red. |
| Label | `label-xs` Inter 12px / 500. Turns `text-error` when the field is invalid. |
| Control | 1px error border. Signed-in: `error` `#ffb4ab`, no extra glow. Auth landing: `status-danger` `#f87171` (peach `#ffb4ab` reads as faded pink next to the CTA) + **3px halo** `rgba(248, 113, 113, 0.22)` on error focus. |
| Helper | `mono-sm` JetBrains Mono 12px / 16px, `text-error`, `role="alert"`. A sentence still uses mono — it is a system message, not body copy. |
| Placement | Directly under the control, or under the last field of a group (`Invalid credentials.`). Never under the page title or description. |
| Gap | `xs` (4px) under a 40px text field. `sm` (8px) under 44px OTP boxes so the 3px error halo does not sit on the text. |
| Empty required | Disable submit. Do not flash “Email and password are required.” |
| Copy | First real field / API message (`Invalid credentials.`, `Invalid authentication code.`). Never Laravel’s wrapper `Validation failed.` |
| Duplicate UI | If the form shows the helper, do not also toast the same string. |

A11y: `aria-invalid` on every invalid control, `aria-describedby` → helper `id`, helper `role="alert"`. Auth landing details (OTP autofill, CTA disable): [auth-landing.md](auth-landing.md).

Checkbox / switch (Atlas pattern):

- Hidden native input
- Box: 14px, radius 3px, rest `outline-variant`, checked fill `primary` with dark check
- Switch: 32×18px track, 12px thumb; on-state emerald tint is allowed for boolean “enabled”

### Badges / tags

Two families:

1. **Status pill** — `rounded-full`, optional 5–6px current-color dot, `mono-sm` or `label-xs` uppercase. Success / warning / danger / unknown.
2. **Row tag** — 4px radius, `label-xs` uppercase, 9–10px. Environment and domain tags (`production`, `staging`, `dev`, `encrypted`, …).

Do not use pill radius on environment tags.

| Tone | Treatment |
|------|-----------|
| ok / up / clean | emerald-400 on emerald-500/15, border 30% |
| warning / degraded / staging | amber-400 on amber-500/15 |
| down / production / failed | red-400 on red-500/20, border 50% |
| unknown / cancelled | `on-surface-variant` on `surface-variant/50` |

### Tables

- Row height 32–36px
- No vertical rules
- 1px `tech-border` under header and each row
- Header: `label-xs`, uppercase, `on-surface-variant`, normal weight
- Primary cell: `body-sm` + `primary`
- Meta under the name: `mono-sm` + `on-surface-variant`
- Numbers / latency: `mono-sm`, right-aligned
- Hover: `layer-2` (`#18181B`)
- Desktop: row actions can appear on hover
- Touch: actions always visible

### Sidebar

- Width 220px, `layer-1`
- Brand row: 48px, logo + `h1` product name (lowercase is fine)
- Group label: `label-xs` uppercase `tracking-widest` `on-surface-variant`
- Item: `body-base`, icon 18px, padding `sm`, radius 4px
- Active: `surface-container` + `primary` text + **2px left `primary` bar** (compensate padding so text does not shift)
- Inactive: `on-surface-variant`, hover `on-surface` + `surface-container`
- Settings / sign-out sit under a top border at the bottom

### Page header

Full-width bar, `layer-1`, `border-b` `tech-border`, padding `md`.

- Title: `h1` / `primary`
- Optional subtitle: `body-sm` / `on-surface-variant`
- Actions right-aligned, `gap-sm`

### Modal / drawer

- Overlay: `black/70`, no blur
- Panel: `layer-2`, `border-tech-bright`, radius 4px, padding `lg`
- Confirm dialogs max ~24rem
- Escape cancels unless a mutation is in flight
- Click overlay cancels
- Destructive confirm uses the destructive button; cancel is secondary

### Toast

Same status colours. No shadow. Border + tinted surface. Bottom or top-right, not blocking the sidebar.

### Terminal / log

- Background `#000000`
- Font `mono-sm` 12px
- Padding 12px
- Scrollbar 4px
- Levels: info muted/blue, warn amber, error rose — keep them in the log, not in chrome

### Skeleton

Pulse rows, `border-b` `tech-border`, no shimmer gradient.

---

## 8. Shell recipe for a new tool

Copy this structure. Change only the product name, logo, and nav items.

1. Root: `flex h-dvh overflow-hidden bg-layer-0 text-on-surface font-sans`
2. Sidebar 220px + main column
3. Main: 48px top bar (`h2` current section) + scrollable `<main>`
4. Pages: optional page header, then `p-md` content
5. One toast host at the shell
6. Mobile: drawer sidebar, menu button in the top bar, Escape / overlay close (no blur)

That is enough for a second tool to feel like Atlas without sharing React.

---

## 9. Tailwind mapping (Atlas)

Atlas maps tokens like this. New tools should do the same so class names stay portable.

```text
bg-layer-0 / bg-layer-1 / bg-layer-2
border-tech / border-tech-bright
text-primary / text-on-surface / text-on-surface-variant
font-h1 text-h1
font-body-sm text-body-sm
font-mono-sm text-mono-sm
bg-primary text-on-primary
bg-status-danger-bg text-status-danger-text border-status-danger-border
w-sidebar_width
gap-xs / gap-sm / gap-md / p-gutter
```

Load fonts:

```text
Inter 400/500/600
JetBrains Mono 400/500
Material Symbols Outlined (FILL 0, wght 300)
```

---

## 10. Out of scope

- Not an npm component library. Spec + tokens only.
- Not ddashboard (different fonts, purple accent, German, desktop-only).
- Not a public marketing brand.
- Do not port Breeze leftovers (`PrimaryButton` gray/indigo, white modal + shadow).

---

## 11. Auth landing (Atlas-only exception)

Unauthenticated Atlas screens (`/login`, `/forgot-password`, `/reset-password`) shipped 2026-11-07 as a **pink branded landing**, not as Technical Precision chrome. That landing is the seed of a possible later app-wide language (pink as chrome, glow, 8–12px radius, 1px pink focus + soft halo). Full snapshot, comparison table, tokens, WCAG 2.1 AA: [auth-landing.md](auth-landing.md).

Until a redesign lands, copy from SPEC for signed-in UI and from `auth-landing.md` only when touching those three routes — or when the user asks to extend the new language. **Field errors** stay `mono-sm` with a coral/red label + border. Signed-in: `error` `#ffb4ab`. Auth landing: `status-danger` `#f87171`. Do not use Inter body or Atlas pink for helpers.
