# Cursor rule export: `min-font-size.mdc`

_Source of agent rule in theme repo: `.cursor/rules/min-font-size.mdc`._

---

# Minimum font-size: 1rem

**All `font-size` values in SCSS must be ≥ 1rem.** 1rem = 14px at the theme base.

## Rule

```scss
/* ✅ correct */
font-size: 1rem;
font-size: 1.1rem;
font-size: 1.286rem;
font-size: 2.5rem;

/* ❌ forbidden — too small */
font-size: 0.875rem;   /* 12.25px */
font-size: 0.8rem;     /* 11.2px  */
font-size: 0.75rem;    /* 10.5px  */
font-size: 0.7rem;     /* 9.8px   */
font-size: 12px;       /* absolute px values < 14px are also forbidden */
font-size: 11px;
```

## Applies to all contexts

- Body text, labels, captions, hints, tooltips
- Button variants (including `.button-small`)
- Form field labels and floating labels
- Table headers and cells
- Meta info rows and sub-headings
- No exceptions for "secondary" or "helper" text

## Exception: badges and pills

Compact status badges and pill indicators may use **`0.857rem`** (12px) as their minimum. These elements are short labels (1–3 words), always paired with full-size text nearby, and benefit from visual compactness.

Selectors that qualify for this exception:
- `.label-ksef`, `.label-portal`
- `*__badge`, `*-badge` (BEM badge elements)
- `*-pill` (status pills)
- Any element whose primary purpose is a compact status/count indicator

## Rationale

Base `font-size: 14px` on `body` (1rem = 14px). Going below 1rem produces text ≤ 12px, which fails WCAG AA readability guidelines on dark backgrounds. All UI text must meet the 1rem floor regardless of context.

## What to use instead of sub-1rem sizes

Use opacity, colour, weight, or letter-spacing to signal hierarchy without shrinking the font:

```scss
/* de-emphasised text — same size, lower opacity colour */
color: $white-40;
font-weight: 300;
letter-spacing: 0.08em;
text-transform: uppercase;

/* compact labels — same size, uppercase + wide tracking */
font-size: 1rem;
text-transform: uppercase;
letter-spacing: 0.12em;
font-weight: 700;
```

## Existing legacy code

Pre-existing SCSS files that contain sub-1rem sizes are **not** auto-fixed by this rule (there are many). When **editing an existing file**, bring touched selectors up to ≥ 1rem as part of the change, except for qualifying badges/pills (see **Exception: badges and pills** above). Do **not** introduce new sub-1rem sizes outside that exception.
