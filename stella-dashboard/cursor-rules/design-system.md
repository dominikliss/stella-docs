# Cursor rule export: `design-system.mdc`

_Source of agent rule in theme repo: `.cursor/rules/design-system.mdc`._

---

# ddashboard Design System

Use this as the single source of truth when building new tools, pages, or components that should look and feel like ddashboard.

---

## Fonts

| Role        | Family          | Notes                                      |
|-------------|-----------------|--------------------------------------------|
| Body / UI   | `"Museo Sans"`  | Default for all text, inputs, buttons      |
| Headings    | `"Nexa Text"`   | `h1`–`h8`, stat numbers (`.big-number`)    |

```css
font-family: "Museo Sans", sans-serif;   /* body default */
font-family: "Nexa Text", sans-serif;    /* headings */
font-size: 14px;                         /* base — 1rem = 14px */
line-height: 1.5;
```

---

## Colour Palette

### Backgrounds (dark hierarchy)
| Token      | Hex       | Use                                            |
|------------|-----------|------------------------------------------------|
| `$black`   | `#131313` | Page / sidebar background                      |
| `$black-2` | `#1E1E1E` | Cards, modals, sidebar panels, toast           |
| `$black-3` | `#2A2A2A` | Borders, dividers, table row separators        |

### Text / Foreground
| Token        | Value                      | Use                              |
|--------------|----------------------------|----------------------------------|
| `$white`     | `#FDFDFD`                  | Primary text, active states      |
| `$white-70`  | `rgba(253,253,253,.7)`     | Secondary text, descriptions     |
| `$white-40`  | `rgba(253,253,253,.4)`     | Placeholder text, subtle borders |
| `$white-30`  | `rgba(253,253,253,.3)`     | Disabled borders                 |
| `$white-20`  | `rgba(253,253,253,.2)`     | Toggle track border              |
| `$white-10`  | `rgba(253,253,253,.1)`     | Hover background on dark buttons |

### Accent Colours
| Token          | Hex / rgba                   | Use                                        |
|----------------|------------------------------|--------------------------------------------|
| `$purple-5`    | `#C266E2`                    | **Primary accent** — links, focus rings, toast accent, toggle on |
| `$purple`      | `#A63FCA`                    | Toggle track when checked                  |
| `$blue-5`      | `#78B2FA`                    | Info, KSeF badge, button-blue              |
| `$blue-5-10`   | `rgba(120,178,250,.1)`       | KSeF badge background                      |
| `$green-5`     | `#6FC773`                    | Success, button-green                      |
| `$red-5`       | `#E14A4D`                    | Error, delete, button-red, toast-error     |
| `$yellow-5`    | `#FBDF62`                    | Warning, button-yellow                     |
| `$pink-5`      | `#F46F8D`                    | button-pink                                |
| `$yellow`      | `#F5C900`                    | Warning icon colour                        |

**Rule:** Never use raw hex values in new SCSS — always reference tokens.

---

## Typography Scale

**Minimum font-size is 1rem everywhere** (see `min-font-size.mdc`). Use colour, weight, tracking, or opacity to convey hierarchy — never shrink text below 1rem.

```
1rem     = 14px   (base — all body, labels, captions, hints, badges)
1.1rem   = 15.4px (slightly prominent body copy)
1.286rem = 18px   (h2 inside details rows)
1.714rem = 24px   (h1 page title)
2.5rem   = 35px   (.big-number stat display)
3rem+            (hero / display headings)
```

Heading font weights follow browser defaults. Body font-weight: 300 for most UI text.

### Hierarchy without shrinking

Instead of using a smaller `font-size`, signal secondary importance with:

```scss
color: $white-40;          /* de-emphasised */
font-weight: 300;          /* lighter weight */
text-transform: uppercase;
letter-spacing: 0.1em;     /* wide tracking for labels/section headings */
```

---

## Layout

### Page Shell
```css
body   { display: flex; background: $black; color: $white; }
#primary {
  flex: 1;
  max-height: 100vh;
  overflow: scroll;
  padding: 20px;
}
```

### Page Header (`.site-meta`)
```html
<div class="site-meta">
  <h1>Page Title</h1>
  <!-- optional: filter inputs on the right -->
</div>
```
`display: flex; justify-content: space-between; align-items: center; padding: 0 0 5px;`

### Content Sections (`.content-section`)
12-column flex grid (out of 12 total). Children use `.content-col-{n}` where n = 2–12.

```html
<div class="content-section">
  <div class="content-col-6">...</div>
  <div class="content-col-6">...</div>
</div>
```

| Class              | Width       |
|--------------------|-------------|
| `.content-col-2`   | ~16.7%      |
| `.content-col-3`   | 25%         |
| `.content-col-4`   | 33.3%       |
| `.content-col-6`   | 50%         |
| `.content-col-8`   | 66.7%       |
| `.content-col-9`   | 75%         |
| `.content-col-12`  | 100%        |

Cards inside `.content-section` get `7.5px` left/right margins (first and last are edge-flush).

---

## Cards

```html
<div class="card">
  <h2>Section title</h2>
  <!-- content -->
</div>
```

```css
.card {
  background-color: $black-2;
  border: 1px solid $black-3;
  border-radius: 4px;
  padding: 20px;
}
```

Card with a header row (title + action button side-by-side):
```html
<div class="card">
  <div class="card-header">
    <h2>Title</h2>
    <button class="button button-dark">Action</button>
  </div>
</div>
```

---

## Buttons

Base class: `.button`
```css
display: inline-flex;
align-items: center;
gap: 8px;
padding: 8px 15px;
border: 1px solid $grey-2;
border-radius: 4px;
font-weight: 300;
background-color: transparent;
transition: all 0.1s ease-out;
```

### Colour Variants
Apply a modifier class alongside `.button`:

| Class           | Border / Text       | Hover bg            | Use                          |
|-----------------|---------------------|---------------------|------------------------------|
| *(none)*        | `$grey-2`           | `$white-10`         | Neutral / default            |
| `.button-dark`  | `$white-40` / `$white` | `$white-10`      | Toolbar actions, dropdowns   |
| `.button-purple`| `$purple-5`         | `$purple-5-10`      | Primary CTA (add, create)    |
| `.button-blue`  | `$blue-5`           | `$blue-5-10`        | Info / secondary action      |
| `.button-green` | `$green-5`          | `$green-5-10`       | Export / success action      |
| `.button-red`   | `$red-5`            | `$red-5-10`         | Delete / destructive         |
| `.button-yellow`| `$yellow-5`         | `$yellow-5-10`      | Warning action               |
| `.button-pink`  | `$pink-5`           | `$pink-5-10`        | Secondary highlight          |

### Size Variants
```css
.button.button-small { padding: 4px 10px; font-size: 1rem; }
.button.small         { padding: 4px 10px; }
```

### CRUD / Icon-only Buttons
```css
.button.button-crud {
  border: none;
  padding: 0;
  margin-left: 10px;
  opacity: 0;                /* hidden by default, shown on row hover */
}
```
Table rows reveal CRUD buttons on `:hover` and `.active`.

### Button Groups
```html
<div class="buttons">
  <button class="button button-dark">…</button>
  <button class="button button-purple">…</button>
</div>
```
`.buttons { display: flex; gap: 10px; }`

### Loading State Inside a Button
Use `<Loader />` component (renders a small spinner) + text:
```jsx
{loading ? <><Loader /> Saving…</> : "Save"}
```
Add `.disabled` class and remove `onClick` while loading.

---

## Data Tables

```html
<div class="data-grid">
  <table>
    <thead>
      <tr><th>Col A</th><th>Col B</th><th></th><th></th></tr>
    </thead>
    <tbody>
      <tr class="active">
        <td>…</td>
        <td class="single-cell">
          <a class="button button-crud button-blue"><IconEdit /></a>
        </td>
        <td class="single-cell">
          <a class="button button-crud button-red"><IconTrash /></a>
        </td>
      </tr>
      <!-- expandable details row -->
      <tr class="details-row active">
        <td colspan="4">
          <ExpandingContent isOpen={…}>…</ExpandingContent>
        </td>
      </tr>
    </tbody>
  </table>
</div>
```

Key rules:
- `th`: `padding: 16px 10px 16px 16px`, `border-bottom: 2px solid $black-3`
- `td`: `padding: 16px 10px 16px 16px`, `border-bottom: 1px solid $black-3`
- Row hover/active: `background-color: $black-2; border-color: $black-3`
- Action columns: class `.single-cell` (width 30px, no padding)
- CRUD buttons are `opacity: 0` at rest, `opacity: 0.8` on row hover/active

---

## Key-Value Data Display (`.data-table`)

```html
<div class="data-table">
  <div class="data-table-row">
    <span class="label">Invoice Date</span>
    <span class="value">2026-02-27</span>
  </div>
</div>
```

```css
.data-table      { display: flex; flex-direction: column; gap: 12px; padding: 10px 20px 20px; }
.data-table-row  { display: flex; gap: 10px; }
.data-table-row .label { font-weight: 700; width: 180px; }
```

---

## Forms

### Structure
```html
<form>
  <div class="field-row">          <!-- groups fields on one line -->
    <div class="field-container">
      <label>Field Name</label>
      <input type="text" />
    </div>
    <div class="field-container">
      <label>Another Field</label>
      <input type="text" />
    </div>
  </div>
</form>
```

### Field array rows (Social links, invoice emails, Kontaktpersonen, etc.)
Each **`.field-array-row`** wraps the main control in **`.field-container`** so labels use the same **floating label** styling as the rest of the form (`<label>` + input, or **`SearchableSelect`** with the **`label`** prop inside `field-container`). Optional leading icon (e.g. edit) and **+/-** buttons sit in the same flex row (`.field-array .field-array-row` in `assets/scss/form.scss`). The Kontaktpersonen block uses **`field-array-outer`** (full width in a `.field-row`) instead of nesting the whole array in an extra `field-container`.

### Field Container
```css
.field-container {
  position: relative;
  margin-bottom: 15px;
  flex: 1;
}
.field-container label {
  /* floats above the border */
  position: absolute;
  top: 10px;
  left: 12px;
  transform: translate(0, -100%);
  font-size: 1rem;
  background-color: $black;     /* must match page/panel bg */
  padding: 3px 5px;
  color: $white;
  font-weight: 300;
}
```

### Input / Select / Textarea
```css
input, select, textarea {
  font-family: "Museo Sans", sans-serif;
  background-color: transparent;
  border: 1px solid $white-40;
  border-radius: 4px;
  color: $white;
  padding: 16px 14px;
  width: 100%;
  font-size: 1rem;
  color-scheme: dark;
  transition: all 0.2s ease-out;
}
input:focus, select:focus, textarea:focus {
  border-color: $purple-5;
  box-shadow: 0 0 0 2px rgba($purple-5, 0.25);
  outline: none;
}
```

### Validation Error
```html
<div class="validation-error">This field is required.</div>
```
`color: $red-5; padding: 10px 14px 0;`

### Sidebar Form (slide-in panel)
Width: `350px`, fixed right side, `background: $black`, `border-left: 1px solid $black-3`, `padding: 20px`, `z-index: 10`.
Animates in/out with `sidebar-slide-in` / `sidebar-slide-out` keyframes (0.25s ease).
Inside a sidebar, `.field-row` stacks vertically (`flex-direction: column`).

---

## Modals

```html
<div class="modal">
  <div class="modal-content">
    <p class="modal-text">Are you sure?</p>
    <div class="modal-actions">
      <a class="button button-red">Delete</a>
      <a class="button">Cancel</a>
    </div>
  </div>
</div>
```

```css
.modal::before { background: rgba(0,0,0,0.6); backdrop-filter: blur(6px); }
.modal-content {
  background: $black-2;
  border: 1px solid $black-3;
  border-radius: 4px;
  padding: 30px 40px;
  max-width: 300px;
}
```

Animations: `modal-backdrop-in` / `modal-content-in` (0.2s ease-out scale + fade). Add `.closing` to trigger reverse.

---

## Dropdowns (`.dropdown`)

```html
<div class="dropdown">
  <button class="button button-dark">Label ▾</button>
  <div class="dropdown-menu">
    <a href="…">Option A</a>
    <a href="…">Option B</a>
  </div>
</div>
```

```css
.dropdown-menu {
  position: absolute;
  top: calc(100% + 8px);
  right: 0;
  background: $black-2;
  border: 1px solid $black-3;
  border-radius: 4px;
  min-width: 220px;
  box-shadow: 0 4px 16px rgba(0,0,0,0.4);
  z-index: 100;
}
.dropdown-menu a { padding: 10px 16px; color: $white; }
.dropdown-menu a:hover { background: $black-3; }
```

---

## Toggle Switch

```jsx
<Toggle checked={value} onChange={fn} label="Enable feature" />
```

Track: `40px × 22px`, `border-radius: 999px`, off = `$black-3`, on = `$purple`.
Thumb: `16px` circle, off = `$white-40`, on = `$white`, shifts `18px` right.

---

## Toast Notifications

```jsx
<ToastContainer toasts={toasts} />
// addToast("Message")  or  addToast("Error", "error")
```

Position: `fixed; bottom: 24px; right: 24px`.
Style: `background: $black-2; border: 1px solid $black-3; border-left: 5px solid $purple-5; border-radius: 4px; padding: 16px 24px; min-width: 280px; box-shadow: 0 4px 16px rgba(0,0,0,.4)`.
Error variant: `border-left-color: $red-5`.
Animates in from right (`toast-slide-in` 0.25s ease-out).

---

## Skeleton Loading

```jsx
import { SkeletonRows } from './skeleton';
// In a table:
{isLoading ? <SkeletonRows rows={7} cols={5} actionCols={2} /> : <ActualRows />}
```

---

## Labels / Badges

Small pill badges next to text:

```html
<span class="label-ksef">KSeF</span>
```

```css
.label-ksef {
  display: inline-block;
  margin-left: 7px;
  padding: 1px 6px;
  border-radius: 4px;
  font-size: 1rem;
  font-weight: 600;
  letter-spacing: 0.04em;
  background-color: $blue-5-10;
  color: $blue-5;
  border: 1px solid rgba(120,178,250,0.3);
  vertical-align: middle;
}
```

For other colours follow the same pattern — use a `*-5-10` background + `*-5` text + matching border.

---

## Icons

Icons live in `src/components/icons.js`. All use `fill="currentColor"` and default to `1em × 1em`.

```jsx
import { IconDownload, IconCreatePdf, IconSendEmail, IconWarning } from './icons';

<IconWarning style={{ color: '#F5C900', marginLeft: '6px', verticalAlign: 'middle' }} />
```

Rules:
- Do **not** set explicit `width` / `height` props — they inherit from font size
- Do **not** add inline `margin` on icons inside `.data-table-row` — use gap on the parent instead

---

## Loader / Spinner

```jsx
import Loader from './loader';
<Loader />
```

Full-page: `40px` diameter, `margin: 200px auto 0`.
Inline (inside `.button`): auto-scaled to `1rem` by the button's nested `.loader` overrides.

---

## Animation Conventions

| Context         | Duration | Easing           |
|-----------------|----------|------------------|
| Hover state     | 0.1s     | `ease-out`       |
| Sidebar slide   | 0.25s    | `ease-out` in, `ease-in` out |
| Modal fade/scale| 0.2s     | `ease-out` in, `ease-in` out |
| Toast slide     | 0.25s in / 0.3s out | `ease-out` / `ease-in` |
| Row transitions | 0.2s     | `ease-out`       |
| Input focus     | 0.2s     | `ease-out`       |

Closing animations always add a `.closing` class and use `forwards` fill mode.

---

## Spacing Reference

| Purpose                        | Value  |
|--------------------------------|--------|
| Page padding                   | 20px   |
| Card padding                   | 20px   |
| Card / section gap             | 7.5px  |
| `.buttons` group gap           | 10px   |
| `.field-row` gap               | 15px   |
| `.field-container` margin-bottom | 15px |
| `.data-table` row gap          | 12px   |
| Table cell padding             | 16px 10px 16px 16px |
| Modal padding                  | 30px 40px |
| Sidebar panel width            | 350px  |
| Toast bottom/right offset      | 24px   |
| Border radius (cards, buttons, inputs) | 4px |

---

## Filter Bar

```html
<div class="filter-bar">
  <div class="filter-bar-dates">
    <input type="date" />
    <span class="filter-bar-sep">–</span>
    <input type="date" />
  </div>
  <div class="filter-bar-shortcuts">
    <a class="button button-small button-dark">This Month</a>
    <a class="button button-small button-dark">Last Month</a>
  </div>
  <div class="filter-bar-actions">
    <!-- push to right with margin-left: auto -->
    <a class="button button-purple">Add Item</a>
  </div>
</div>
```

`.filter-bar { display: flex; align-items: center; gap: 12px; padding: 12px 0 16px; flex-wrap: wrap; }`
Actions section: `margin-left: auto` to right-align.
