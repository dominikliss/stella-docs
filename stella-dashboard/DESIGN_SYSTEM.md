# ddashboard — Design System Brief

> Self-contained reference for a design agent. All values are inlined. No external files need to be consulted.

---

## PROJECT OVERVIEW

**ddashboard** is a private, desktop-only business operations dashboard built as a WordPress theme. It is **not** a public website — it is an internal admin tool for one operator to manage clients, invoices, transactions, accounting, and integrations.

- **Audience:** Single operator (private, German-language)
- **Viewport:** Desktop only — minimum ~1200px wide. No mobile breakpoints.
- **Tone:** Clean, professional, data-dense, dark
- **Language:** German (all labels and navigation)
- **Technology:** WordPress (PHP) + React (compiled JS), SCSS compiled server-side

---

## BRAND INFORMATION

- **Owner:** Dominik Liss, BSc
- **Logo:** White SVG wordmark/logomark — displayed in the sidebar at 55×55px on `#131313` background
- **Primary accent:** Purple (`#C266E2`) — the single dominant interactive color
- **No public brand guidelines** — the aesthetic is dark SaaS application, not a marketing brand

---

## COLOR SYSTEM

### Background Scale

| Role | Hex | Usage |
|---|---|---|
| Deepest / base | `#131313` | Page background, sidebar, bottom surface |
| Card / elevated | `#1E1E1E` | Cards, modals, active rows, sidebar form background |
| Border / hover | `#2A2A2A` | All borders, dividers, subtle hover states |

### Text Scale

| Role | Hex / Value | Usage |
|---|---|---|
| Primary text | `#FDFDFD` | All body text, headings, inputs |
| Secondary text | `rgba(253,253,253,0.7)` | Nav links, helper text |
| Muted text | `rgba(253,253,253,0.4)` | Disabled borders, sub-values |
| Very muted | `rgba(253,253,253,0.1)` | Hover tints on dark surfaces |
| Hint / description | `#8E8E8E` | Field hints, stat subtitles, meta info |
| Default button border | `#DEDEDE` | Default (uncolored) button border + text |

### Semantic Color Palette

Every semantic color has 5 shades. The UI always uses **shade 5** (the lightest, most vibrant) for text/borders and **shade 5-10** (10% alpha of shade 5) for hover backgrounds.

#### Red — Destructive actions, errors
| Shade | Hex |
|---|---|
| 1 (darkest) | `#C0282B` |
| 2 | `#851C1B` |
| 3 | `#6E1A18` |
| 4 | `#5E1412` |
| **5 (UI color)** | **`#E14A4D`** |
| 5-10 (hover bg) | `rgba(225,74,77,0.1)` |

#### Green — Income, success, confirmed
| Shade | Hex |
|---|---|
| 1 | `#4DB152` |
| 2 | `#409344` |
| 3 | `#337C37` |
| 4 | `#295F2C` |
| **5 (UI color)** | **`#6FC773`** |
| 5-10 (hover bg) | `rgba(111,199,115,0.1)` |

#### Blue — Info, edit actions, download, links
| Shade | Hex |
|---|---|
| 1 | `#609EE9` |
| 2 | `#1858A8` |
| 3 | `#164B8E` |
| 4 | `#143660` |
| **5 (UI color)** | **`#78B2FA`** |
| 5-10 (hover bg) | `rgba(120,178,250,0.1)` |

#### Yellow — Warnings, preview actions
| Shade | Hex |
|---|---|
| 1 | `#F5C900` |
| 2 | `#D7B724` |
| 3 | `#CCAC1F` |
| 4 | `#C1A217` |
| **5 (UI color)** | **`#FBDF62`** |
| 5-10 (hover bg) | `rgba(251,223,98,0.1)` |

#### Pink — Tertiary accent (rare)
| Shade | Hex |
|---|---|
| 1 | `#E94C6F` |
| 2 | `#BE3C59` |
| 3 | `#9C354C` |
| 4 | `#943248` |
| **5 (UI color)** | **`#F46F8D`** |
| 5-10 (hover bg) | `rgba(244,111,141,0.1)` |

#### Purple — PRIMARY ACCENT (active states, CTA, focus, branding)
| Shade | Hex |
|---|---|
| 1 | `#A63FCA` |
| 2 | `#8637A1` |
| 3 | `#6E2D85` |
| 4 | `#622876` |
| **5 (UI color)** | **`#C266E2`** |
| 5-10 (hover bg) | `rgba(194,102,226,0.1)` |

### Where Purple Is Used
- Active navigation link: text `#C266E2`, background `rgba(194,102,226,0.1)`
- Focused form input: `border-color: #C266E2; box-shadow: 0 0 0 2px rgba(#C266E2,0.25)`
- Primary action buttons (`.button-purple`)
- Toast left border (`border-left: 5px solid #C266E2`)
- Sidebar template-edit left border (`border-left: 3px solid #C266E2`)
- Tab count badges (background `#A63FCA`)
- Hyperlinks: `color: #C266E2; opacity: 0.8` (hover `opacity: 1`)

---

## TYPOGRAPHY

### Font Families

| Family | Role | Fallback |
|---|---|---|
| **Museo Sans** | Body, buttons, inputs, nav, prose | `sans-serif` |
| **Nexa Text** | Headings (h1–h8), big number displays | `sans-serif` |

Both fonts are self-hosted. Museo Sans handles everything functional. Nexa Text gives headings weight and character.

### Base Settings

| Property | Value |
|---|---|
| Base font size | `14px` |
| Line height | `1.5` |
| Body color | `#FDFDFD` |
| Body background | `#131313` |

### Type Scale

| Element / Context | Size | Weight | Font | Notes |
|---|---|---|---|---|
| `h1` (page title) | `1.714rem` (~24px) | — | Nexa Text | flex + align-items center |
| `h2` (section / card) | `1.286rem` (~18px) | — | Nexa Text | — |
| `h2` (management sections) | `1rem` | 700 | Nexa Text | UPPERCASE, letter-spacing 0.04em |
| `h2` (stat section titles) | `1rem` | 700 | Nexa Text | UPPERCASE, letter-spacing 0.04em |
| Nav links | `1.143rem` | 500 | Museo Sans | — |
| Buttons | `1rem` | 300 | Museo Sans | — |
| Body text / table cells | `1rem` | normal | Museo Sans | — |
| Field labels (floating) | `0.857rem` | 300 | Museo Sans | — |
| Field description / hints | `0.857rem` | — | Museo Sans | color `#8E8E8E` |
| Small buttons | `0.8rem` | — | Museo Sans | — |
| Badge labels (KSeF, Portal) | `0.7rem` | 600 | Museo Sans | UPPERCASE, letter-spacing 0.04em |
| Tab count badge | `0.72rem` | 600 | Museo Sans | — |
| Toast messages | `1.1rem` | — | Museo Sans | — |
| KPI labels | `0.75rem` | 600 | Museo Sans | UPPERCASE, letter-spacing 0.06em |
| KPI values | `1.35–1.45rem` | 700 | Museo Sans | — |
| KPI sub-values | `0.8rem` | — | Museo Sans | color `rgba(253,253,253,0.4)` |
| Big number display | `2.5rem` | — | Nexa Text | `.big-number` class |

---

## LAYOUT & SHELL

### Application Shell

```
┌─────────────────────────────────────────────────────────────────┐
│  <header>  250px fixed       │  <main id="primary">            │
│  height: 100vh               │  flex: 1                        │
│  background: #131313         │  max-height: 100vh              │
│  border-right: 1px #2A2A2A   │  overflow: scroll               │
│  padding: 40px 12px          │  padding: 20px                  │
│                              │                                 │
│  Logo 55×55px (left, 25px)   │  <article.hentry>               │
│  margin-top: 60px for nav    │  padding: 20px 0                │
│                              │                                 │
│  Nav links (see below)       │  [Page content / React app]     │
└──────────────────────────────┴─────────────────────────────────┘
```

- `body { display: flex }` — sidebar and main sit side-by-side
- WordPress admin bar is hidden
- All content scrolls within `#primary`, not the body

### Page Header (`.site-meta`)

Every page has a top bar:

```
┌──────────────────────────────────────────────────────────┐
│  <h1>Page Title</h1>      │  [filters]  [action buttons] │
│  (left aligned)           │  (right aligned)             │
└──────────────────────────────────────────────────────────┘
```

- `display: flex; align-items: center; justify-content: space-between; padding: 0 0 5px`

### 12-Column Grid (`.content-section`)

Used inside page content for side-by-side layouts:

| Class | Width |
|---|---|
| `.content-col-2` | 16.67% |
| `.content-col-3` | 25% |
| `.content-col-4` | 33.33% |
| `.content-col-5` | 41.67% |
| `.content-col-6` | 50% |
| `.content-col-7` | 58.33% |
| `.content-col-8` | 66.67% |
| `.content-col-9` | 75% |
| `.content-col-10` | 83.33% |
| `.content-col-11` | 91.67% |
| `.content-col-12` | 100% |

---

## COMPONENTS

### Navigation (Sidebar)

```
padding: 12px 23px
border-radius: 8px
margin-bottom: 5px
font-size: 1.143rem
font-weight: 500
font-family: Museo Sans
transition: all 0.2s ease-out
```

| State | Text color | Background |
|---|---|---|
| Default | `rgba(253,253,253,0.7)` | transparent |
| Hover | `rgba(253,253,253,0.7)` | `#2A2A2A` |
| Active | `#C266E2` | `rgba(194,102,226,0.1)` |

### Tabs (`.tabs`)

Horizontal tab strip above content sections.

```
display: flex; gap: 8px; margin-bottom: 8px
```

Each tab:
```
padding: 8px 15px
border-radius: 4px (top only in older usage)
border-bottom: 2px solid transparent
color: #FDFDFD
opacity: 0.5 (inactive)
```

| State | opacity | background | border-bottom |
|---|---|---|---|
| Inactive | 0.5 | transparent | transparent |
| Active | 1 | `#2A2A2A` | `2px solid #DEDEDE` |

### Buttons

**Base `.button`:**
```css
display: inline-flex;
align-items: center;
gap: 8px;
padding: 8px 15px;
border-radius: 4px;
border: 1px solid #DEDEDE;
color: #DEDEDE;
background: transparent;
font-weight: 300;
font-size: 1rem;
transition: all 0.1s ease-out;
```
Hover: `background: rgba(253,253,253,0.1); color: #FDFDFD`

**Size variant `.button-small`:** `padding: 4px 10px; font-size: 0.8rem`

**Color variants** — each changes border + text to shade 5, hover bg to shade 5-10:

| Class | Border / Text | Hover Background |
|---|---|---|
| `.button-red` | `#E14A4D` | `rgba(225,74,77,0.1)` |
| `.button-green` | `#6FC773` | `rgba(111,199,115,0.1)` |
| `.button-blue` | `#78B2FA` | `rgba(120,178,250,0.1)` |
| `.button-yellow` | `#FBDF62` | `rgba(251,223,98,0.1)` |
| `.button-pink` | `#F46F8D` | `rgba(244,111,141,0.1)` |
| `.button-purple` | `#C266E2` | `rgba(194,102,226,0.1)` |
| `.button-dark` | border `rgba(253,253,253,0.4)`, text `#FDFDFD` | `rgba(253,253,253,0.1)` |

**CRUD icon buttons (`.button-crud`)** — used in table rows for actions (edit, delete, download, preview):
```css
display: inline-flex;
align-items: center;
justify-content: flex-start;
border: none;
padding: 0;
margin-left: 10px;
color: #FDFDFD;
opacity: 0;              /* hidden at rest */
background: transparent;
```
- Icon size: 14×14px SVG
- On row hover: `opacity: 0.8`
- Colors: same `-red/-blue/-green/-yellow/-purple/-pink` class names, but only text color changes (no border)

### Tables

```css
border-collapse: collapse;
width: 100%;
margin-top: 15px;
```

`th`:
```css
text-align: left;
padding: 16px 10px 16px 16px;
border-bottom: 2px solid #2A2A2A;
border-top: 1px solid #2A2A2A;
font-weight: 700;
```

`td`:
```css
padding: 16px 10px 16px 16px;
border-bottom: 1px solid #2A2A2A;
```

`tbody tr`:
```css
background: transparent;
transition: all 0.2s ease-out;
border-left: 1px solid transparent;
border-right: 1px solid transparent;
cursor: pointer;
```

| State | background | border |
|---|---|---|
| Default | transparent | transparent |
| Hover | `#1E1E1E` | `#2A2A2A` |
| Active (expanded) | `#1E1E1E` | `#2A2A2A`, no bottom |

**Expandable detail row (`.details-row`):**
- `td { padding: 0; border-bottom: none }`
- Inner content: `overflow: hidden; transition: max-height 0.3s ease-in-out`
- When active: `border-bottom: 1px solid #2A2A2A`

**Single-cell column (`.single-cell`):** `width: 30px; padding: 0` — used for icon-only CRUD columns

**Logo cell (`.logo-cell`):** `width: 25px` — 25×25px client logo thumbnail

### Cards (`.card`)

```css
display: inline-block;
background: #1E1E1E;
border: 1px solid #2A2A2A;
border-radius: 4px;
padding: 20px;
```

- `.card > h2 { margin-top: 0 }`
- `.card-header { display: flex; align-items: center; justify-content: space-between }`
- Max-width variants per context: products / accounting-configs cards → `width: 400px`

### Forms

**Floating label pattern (`.field-container`):**
```css
position: relative;
margin-bottom: 15px;
flex: 1;
```

Label:
```css
position: absolute;
transform: translate(0, -100%);
top: 10px;
left: 12px;
font-size: 0.857rem;
font-weight: 300;
color: #FDFDFD;
background: #131313;    /* or #1E1E1E inside a card */
padding: 3px 5px;
border-radius: 2px;
line-height: 1.2;
z-index: 1;
```

All inputs (`text, email, url, password, date, number, select, textarea`):
```css
font-family: "Museo Sans", sans-serif;
background: transparent;
border: 1px solid rgba(253,253,253,0.4);
border-radius: 4px;
color: #FDFDFD;
padding: 16px 14px;
width: 100%;
font-size: 1rem;
color-scheme: dark;
transition: all 0.2s ease-out;
```

Focus state:
```css
border-color: #C266E2;
box-shadow: 0 0 0 2px rgba(194,102,226,0.25);
outline: none;
```

Select: Custom SVG chevron background image, `-webkit-appearance: none`

**Field row (`.field-row`):** `display: flex; gap: 15px; margin-bottom: 15px`

**Validation error (`.validation-error`):** `color: #E14A4D; padding: 10px 14px 0`

**Submit button:** `font-family: Museo Sans; font-size: 1rem; margin-top: 15px`

**Field hint (`.field-description`):** `font-size: 0.857rem; color: rgba(253,253,253,0.7); padding: 8px 14px 0`

### Sidebar Form (`.sidebar-form`)

Slide-in panel from the right side of the viewport:

```css
position: fixed;
top: 0; right: 0; bottom: 0;
width: 350px;
z-index: 10;
background: #131313;
border-left: 1px solid #2A2A2A;
padding: 20px;
overflow: scroll;
animation: sidebar-slide-in 0.25s ease-out;
```

- Wide variant: `width: min(720px, 96vw)` — used for file/document forms
- Template-edit stacked variant: `z-index: 30; border-left: 3px solid #C266E2; box-shadow: -8px 0 32px rgba(0,0,0,0.45)`
- Close button: top-right corner, `font-size: 1.5rem`

Keyframes:
```css
@keyframes sidebar-slide-in {
  from { transform: translateX(100%); opacity: 0; }
  to   { transform: translateX(0);    opacity: 1; }
}
@keyframes sidebar-slide-out {
  from { transform: translateX(0);    opacity: 1; }
  to   { transform: translateX(100%); opacity: 0; }
}
```

### Modals

```css
position: fixed;
inset: 0;
z-index: 9;
display: flex;
align-items: center;
justify-content: center;
```

Backdrop (`:before` pseudo-element):
```css
position: absolute; inset: 0;
background: rgba(0,0,0,0.6);
backdrop-filter: blur(6px);
-webkit-backdrop-filter: blur(6px);
```

Modal content box:
```css
background: #1E1E1E;
color: #FDFDFD;
padding: 30px 40px;
border: 1px solid #2A2A2A;
border-radius: 4px;
max-width: 300px;
display: flex;
flex-direction: column;
gap: 15px;
animation: modal-content-in 0.2s ease-out;
```

Email modal variant: `width: 520px; max-width: 520px`

Keyframes:
```css
@keyframes modal-content-in {
  from { transform: scale(0.92); opacity: 0; }
  to   { transform: scale(1);    opacity: 1; }
}
@keyframes modal-content-out {
  from { transform: scale(1);    opacity: 1; }
  to   { transform: scale(0.92); opacity: 0; }
}
```

### Toasts (`.toast-container` / `.toast`)

Container:
```css
position: fixed;
bottom: 24px; right: 24px;
z-index: 1000;
display: flex; flex-direction: column; gap: 10px;
pointer-events: none;
```

Toast item:
```css
display: flex; align-items: center; gap: 14px;
background: #1E1E1E;
border: 1px solid #2A2A2A;
border-left: 5px solid #C266E2;    /* purple accent */
color: #FDFDFD;
padding: 16px 24px;
border-radius: 4px;
min-width: 280px;
font-size: 1.1rem;
line-height: 1.3;
box-shadow: 0 4px 16px rgba(0,0,0,0.4);
animation: toast-slide-in 0.25s ease-out;
```

Error variant: `border-left-color: #E14A4D`

```css
@keyframes toast-slide-in {
  from { transform: translateX(110%); opacity: 0; }
  to   { transform: translateX(0);    opacity: 1; }
}
@keyframes toast-slide-out {
  from { transform: translateX(0);    opacity: 1; }
  to   { transform: translateX(110%); opacity: 0; }
}
```

### Skeleton Loading States

```css
.skeleton-block {
  height: 14px;
  border-radius: 3px;
  background: linear-gradient(
    90deg,
    #2A2A2A 25%,
    rgba(255,255,255,0.06) 50%,
    #2A2A2A 75%
  );
  background-size: 1200px 100%;
  animation: skeleton-shimmer 1.8s ease-in-out infinite;
}
@keyframes skeleton-shimmer {
  0%   { background-position: -600px 0; }
  100% { background-position:  600px 0; }
}
```

Rows stagger by 0.08s each. Data rows appear with:
```css
@keyframes row-appear {
  from { opacity: 0; transform: translateY(5px); }
  to   { opacity: 1; transform: translateY(0); }
}
/* animation: row-appear 0.22s ease-out both */
```

### Loader (Spinner)

```css
.loader {
  width: 40px; aspect-ratio: 1;
  border-radius: 50%;
  border: 5px solid transparent;
  border-right-color: #FDFDFD;
  animation: l24 1s infinite linear;
  margin: 200px auto 0;
}
/* :before → 2s; :after → 4s — three concentric rings */
@keyframes l24 { 100% { transform: rotate(1turn); } }
```

### Badge Labels

**KSeF label (`.label-ksef`):**
```css
display: inline-block;
background: rgba(120,178,250,0.1);
color: #78B2FA;
border: 1px solid rgba(120,178,250,0.3);
font-size: 0.7rem; font-weight: 600;
letter-spacing: 0.04em;
border-radius: 4px;
padding: 1px 6px;
```

**Portal label (`.label-portal`):**
```css
background: rgba(194,102,226,0.1);
color: #C266E2;
border: 1px solid rgba(194,102,226,0.35);
/* same sizing as KSeF */
```

**Tab count badge (`.tab-badge`):**
```css
display: inline-flex; align-items: center; justify-content: center;
margin-left: 7px;
min-width: 18px; height: 18px; padding: 0 5px;
border-radius: 9px;
background: #A63FCA;  /* purple shade 1 */
color: #FDFDFD;
font-size: 0.72rem; font-weight: 600;
```

### Key-Value Data Display (`.data-table`)

```css
display: flex; flex-direction: column; gap: 12px;
padding: 10px 20px 20px;
```

Each row:
```css
display: flex; gap: 10px;
/* .label { font-weight: 700; width: 180px } */
/* .value { flex: 1 } */
```

### Data List (`.data-list` / `.data-list-item`)

Compact list with hover-revealed actions (used for files/documents per client):
```css
.data-list-item {
  padding: 0 12px;
  border-radius: 4px;
  display: flex; gap: 10px;
  justify-content: space-between;
}
/* Hover: background #2A2A2A */
/* .actions: hidden by default, display:flex on hover */
/* .label: padding 6px 0, flex:1 */
```

### Accounting KPI Cards (`.accounting-stats-kpi`)

```css
.accounting-stats-kpi-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));
  gap: 16px;
  margin-bottom: 36px;
}
.accounting-stats-kpi {
  border: 1px solid #2A2A2A;
  border-radius: 8px;
  padding: 16px 18px;
  background: #1E1E1E;
}
.accounting-stats-kpi-label {
  font-size: 0.75rem; font-weight: 600;
  letter-spacing: 0.06em; text-transform: uppercase;
  color: #8E8E8E; margin-bottom: 10px;
}
.accounting-stats-kpi-value {
  font-size: 1.35rem; font-weight: 700; color: #FDFDFD;
}
.accounting-stats-kpi-sub {
  margin-top: 8px; font-size: 0.8rem; color: rgba(253,253,253,0.4);
}
```

KPI semantic value colors:
- Income: `#6FC773` (green-5)
- Expense: `#E14A4D` (red-5)
- Warning: `#FBDF62` (yellow-5)
- Investment: `#78B2FA` (blue-5)
- Bank/summary: `#C266E2` (purple-5), value size `1.45rem`

### Management Section Cards (`.management-section`)

```css
display: flex; flex-direction: column; gap: 24px;
padding: 24px;
border: 1px solid #2A2A2A;
border-radius: 8px;

h2 {
  font-size: 1rem; font-weight: 700;
  text-transform: uppercase; letter-spacing: 0.04em;
  color: #FDFDFD; margin: 0;
}
```

---

## ICONS

All icons are inline SVG (`currentColor`, default `1em×1em`). In table CRUD context: `14×14px`.

| Icon | Visual | Color context |
|---|---|---|
| Preview / open in tab | Square with arrow pointing out | Yellow (`.button-yellow`) |
| Download | Downward arrow + tray | Green (`.button-green`) |
| Create PDF | Document with plus | Purple (`.button-purple`) |
| Edit | Pencil | Blue (`.button-blue`) |
| Delete | Trash can | Red (`.button-red`) |
| Send email | Envelope | Default white |
| Warning | Triangle + exclamation | Yellow |

---

## ANIMATION REFERENCE

| Element | Duration | Easing | Property |
|---|---|---|---|
| Button / link hover | 100ms | ease-out | color, background |
| Nav link hover | 200ms | ease-out | background, color |
| Table row hover | 200ms | ease-out | background |
| Detail row expand | 300ms | ease-in-out | max-height |
| Sidebar slide-in | 250ms | ease-out | translateX + opacity |
| Sidebar slide-out | 250ms | ease-in | translateX + opacity |
| Modal backdrop in/out | 200ms | ease-out / ease-in | opacity |
| Modal content in/out | 200ms | ease-out / ease-in | scale + opacity |
| Toast slide-in | 250ms | ease-out | translateX + opacity |
| Toast slide-out | 300ms | ease-in | translateX + opacity |
| Row appear | 220ms | ease-out | opacity + translateY 5px |
| Skeleton shimmer | 1800ms | ease-in-out | background-position (loop) |

---

## SPACING REFERENCE

| Context | Value |
|---|---|
| Sidebar padding | `40px 12px` |
| Logo left offset | `25px` |
| Nav margin-top | `60px` |
| Nav link padding | `12px 23px` |
| Nav link border-radius | `8px` |
| Main content padding | `20px` |
| Article padding | `20px 0` |
| Button padding (default) | `8px 15px` |
| Button padding (small) | `4px 10px` |
| Button gap (icon+text) | `8px` |
| Card padding | `20px` |
| Card border-radius | `4px` |
| Management section padding | `24px` |
| Management section border-radius | `8px` |
| KPI card padding | `16px 18px` |
| KPI card border-radius | `8px` |
| Form field margin-bottom | `15px` |
| Form input padding | `16px 14px` |
| Field row gap | `15px` |
| Table th/td padding | `16px 10px 16px 16px` |
| Modal content padding | `30px 40px` |
| Modal max-width (default) | `300px` |
| Email modal width | `520px` |
| Toast padding | `16px 24px` |
| Toast min-width | `280px` |
| Sidebar form padding | `20px` |
| Sidebar form width (default) | `350px` |
| Sidebar form width (wide) | `min(720px, 96vw)` |
| Data table gap | `12px` |
| KPI grid gap | `16px` |
| Buttons group gap | `10px` |

---

## APPLICATION PAGES

Each page = one WordPress post with a shortcode that mounts a React app into an empty `<div>`.

| Page slug | Shortcode | Purpose |
|---|---|---|
| `/buchhaltung` | `[stella_accounting]` | Main accounting: date filter, line chart, income/expense transaction table with KSeF import |
| `/kunden` | `[stella_clients]` | Client list → expand row → client card (tabs: data / contacts / files / reports / portal) |
| `/rechnungen` | `[stella_invoices]` | Invoice list with PDF generation, download, preview, send email |
| `/offene-rechnungen` | `[stella_open_invoices]` | Invoices with no payment date yet |
| `/produkte` | `[stella_products]` | Product cards, sidebar create/edit form |
| `/buchhaltung-jahre` | `[stella_accounting_configs]` | Per-year accounting config cards (budgets, VAT quarters, insurance) |
| `/statistiken` | `[stella_accounting_stats]` | Financial KPI dashboard — income, taxes, investment, bank targets |
| `/verwaltung` | `[stella_management_page]` | Integration settings: KSeF, Saldeo, YouTube, AI |
| `/marketing` | virtual route | Marketing module + AI chat panel (Claude) |
| `/login` | (WP login page) | Login form styled to match the theme |

---

## DATA MODEL (Custom Post Types)

| Post Type | Purpose | Key fields |
|---|---|---|
| `client` | Business clients | `official_name`, `email`, `invoice_emails[]`, `address`, `vat_number`, `people[]` (linked persons), `status` (active/lead/inactive), `website`, `logo`, portal user flag |
| `person` | Contact persons | `name`, `email`, `phone`, `birthday`, `profile_image` — linked to one or more clients |
| `transaction` | Income / expense | `type` (Einnahme/Ausgabe), `price_net`, `vat_percent`, `currency`, `invoice_date`, `transaction_date`, `note`, `internal_category`, `ksef_reference` |
| `invoice` | Invoice metadata | `transaction_id`, `number`, `invoice_date`, `invoice_url`, `positions[]`, `language_code`, `client_id` |
| `product` | Products / subscriptions | `client_id`, `price_net`, `vat_percent`, subscription frequency fields |
| `accountingconfig` | Per-year settings | `year`, `is_completed`, `vacation`, `sick_days`, `monthly_investment_budget`, `expenses_budget`, `emergency_fund`, `income_tax_paid`, `open_sv_contribution`, `vat_[1–4]_quarter_paid`, `open_costs` |
| `file` | Proposals / contracts / docs | `type` (proposal/contract/document), `client_id`, `is_template`, `public_url`, `document_type` (normal / terms_of_service) |
| `credentials` | Internal credentials store | — |

---

## PDF VISUAL DESIGN

Invoices and reports are generated as PDFs with a shared dark header matching the UI theme.

### Header Bar

```
background: #131313
padding: 45px 50px 30px
three columns: [Customer info] | [Profile photo 85×85px] | [Dominik info]
```

- All header text: `#FDFDFD`
- Logo: white SVG, ~37×37px
- Font faces: MuseoSans (body), NexaTextBlack / NexaTextHeavy (display)
- Below header: document title (left) + date (right)

### Page Setup

- Page size: A4 portrait
- Margins (most PDFs): `13mm` all sides
- Invoices: `0mm` margins (header spans edge-to-edge, inner content has `50px` horizontal padding)

---

## DESIGN PRINCIPLES

1. **Dark-first, always.** The application is designed from the ground up for dark mode. `#131313` is the lowest layer. Never use white or light backgrounds in the application shell.

2. **Single accent color — purple.** Do not introduce competing accent colors in UI chrome. Purple (`#C266E2`) is the one interactive/active color. All other colors are semantic: red = destructive, green = income/success, yellow = warning/preview, blue = edit/info.

3. **Data density over decoration.** Tables are the primary layout surface. Preserve readable type and adequate padding but resist decorative whitespace.

4. **Inline affordances, hidden until hover.** CRUD actions (edit, delete, download, preview) live in the table row and are `opacity: 0` until the row is hovered. This keeps the table clean.

5. **Slide-in panels, not page navigation.** All create and edit flows happen in a `350px` sidebar panel sliding from the right. The list behind it stays visible and in context.

6. **Floating labels on all inputs.** Labels sit on the border, floating above the field. Background color of the label must match its parent surface (`#131313` or `#1E1E1E`) to appear to cut into the border.

7. **Toasts for all async feedback.** Save success, errors, import results — all appear as toasts bottom-right. No inline success states on buttons.

8. **Fast, directional animations.** All transitions are 100–300ms, ease-out entry, ease-in exit. No bouncing, no spring physics. The app feels snappy and responsive.

9. **German language throughout.** All UI labels, navigation, and feedback messages are in German. Any redesign must preserve this.

10. **Shortcodes as mount points.** Each WP page renders `<div class="dls-[app]"></div>`. React mounts there. A redesign can replace SCSS while keeping HTML + React structure intact.
