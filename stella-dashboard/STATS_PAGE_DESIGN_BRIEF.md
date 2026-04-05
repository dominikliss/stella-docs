# Statistics Page — Design Brief for Redesign

> Fully self-contained. All data, content, layout, styling, and logic is described here.
> Copy-paste this entire file into your design AI tool.

---

## WHAT THIS PAGE IS

**Page title:** Buchhaltung — Übersicht (German: "Accounting — Overview")
**Route:** `/statistiken`
**Purpose:** A financial dashboard for a freelance/agency owner. It shows income, expenses, tax obligations, investment budget status, vacation/sick-day reserves, and a total "money needed on the bank account" liquidity figure — all computed from real transaction data + per-year configuration values.

This is a **private internal tool** — dark theme, desktop only (minimum ~1200px wide), no mobile layout needed. All text is in **German**.

---

## BRAND & VISUAL SYSTEM

### Colors (exact hex — use these)

| Token | Hex | Role |
|---|---|---|
| Page background | `#131313` | Deepest surface |
| Card / elevated | `#1E1E1E` | KPI cards, section surfaces |
| Border / divider | `#2A2A2A` | All borders |
| Primary text | `#FDFDFD` | Headings, values, labels |
| Secondary text | `rgba(253,253,253,0.7)` | Subtitles |
| Muted / hint text | `#8E8E8E` | KPI labels, section descriptions, meta |
| Muted value | `rgba(253,253,253,0.4)` | KPI sub-values |
| **Purple (primary accent)** | **`#C266E2`** | Bank/liquidity KPI value, accent |
| Green (income) | `#6FC773` | Income KPI value |
| Red (expense) | `#E14A4D` | Expense KPI value |
| Yellow (warning) | `#FBDF62` | Tax/insurance KPI value, warning text |
| Blue (investment) | `#78B2FA` | Investment budget KPI value |

### Typography

| Element | Font | Size | Weight | Style |
|---|---|---|---|---|
| Page heading | Nexa Text | 1.286rem (~18px) | normal | — |
| Meta line (date/year) | Museo Sans | 0.9rem | normal | color `#8E8E8E` |
| Warning text | Museo Sans | 0.9rem | normal | color `#FBDF62` |
| KPI label | Museo Sans | 0.75rem | 600 | UPPERCASE, letter-spacing 0.06em |
| KPI value | Museo Sans | 1.35rem | 700 | varies by color |
| KPI value (bank) | Museo Sans | 1.45rem | 700 | color `#C266E2` |
| KPI sub-text | Museo Sans | 0.8rem | normal | color `rgba(253,253,253,0.4)` |
| Section title | Museo Sans | 1rem | 700 | UPPERCASE, letter-spacing 0.04em |
| Section description | Museo Sans | 0.85rem | normal | color `#8E8E8E`, max-width 720px |
| Data row label | Museo Sans | 1rem | 700 | width 180px |
| Data row value | Museo Sans | 1rem | normal | — |
| Emphasis row value | Museo Sans | 1.25rem | 700 | color `#FDFDFD` |
| Loading text | Museo Sans | 1rem | normal | color `#8E8E8E` |

---

## PAGE STRUCTURE (top to bottom)

```
┌─────────────────────────────────────────────────────────────────┐
│  SIDEBAR (250px, not part of this page)                        │
├─────────────────────────────────────────────────────────────────┤
│  MAIN CONTENT AREA  (padding: 20px, scrollable)                │
│                                                                 │
│  ┌──── PAGE HEADER ────────────────────────────────────────┐   │
│  │  h2: "Buchhaltung — Übersicht"                          │   │
│  │  meta: "Kalenderjahr 2026, Monat 3 · Datenstand: …"    │   │
│  │  [optional warning if no config for this year]          │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌──── KPI GRID (6 cards) ─────────────────────────────────┐   │
│  │  [Card 1] [Card 2] [Card 3] [Card 4] [Card 5] [Card 6] │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌──── SECTION 1: Config ──────────────────────────────────┐   │
│  │  SECTION TITLE (uppercase)                              │   │
│  │  section description (muted)                            │   │
│  │  data rows (label | value pairs, max-width 800px)       │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌──── SECTION 2: Income & Transactions ───────────────────┐   │
│  │  … same pattern …                                       │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  (sections 3–7 follow the same pattern, listed below)          │
│                                                                 │
│  ┌──── SECTION 7: Liquidity (EMPHASIS) ───────────────────┐   │
│  │  One large emphasis row: label | big bold value         │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

---

## 1. PAGE HEADER

**HTML class:** `.accounting-stats-header` (uses `.site-meta` base)

```
h2: "Buchhaltung — Übersicht"
p.accounting-stats-meta: "Kalenderjahr 2026, Monat 3 · Datenstand: 2026-03"
```

Optional warning (shown in yellow `#FBDF62` if no accounting config exists for the current year):
```
"— Kein Buchhaltungs-Jahr 2026 in den Einstellungen (Budgets & Soll-Werte fehlen)"
```

**Styling:**
- margin-bottom: `28px`
- h2 margin-bottom: `8px`
- Meta text: `0.9rem`, color `#8E8E8E`

---

## 2. KPI GRID

**Layout:** CSS Grid, `repeat(auto-fill, minmax(200px, 1fr))`, gap `16px`, margin-bottom `36px`

6 KPI cards displayed in one responsive row (on a 1200px+ desktop they all fit in one row).

### Card anatomy:

```
┌────────────────────────────────┐
│  LABEL (small, muted, caps)    │
│                                │
│  VALUE (large, bold, colored)  │
│                                │
│  sub-text (tiny, muted)        │
└────────────────────────────────┘
```

Card styling:
- background: `#1E1E1E`
- border: `1px solid #2A2A2A`
- border-radius: `8px`
- padding: `16px 18px`

Label styling:
- font-size: `0.75rem`
- font-weight: `600`
- text-transform: UPPERCASE
- letter-spacing: `0.06em`
- color: `#8E8E8E`
- margin-bottom: `10px`

Value styling:
- font-size: `1.35rem`
- font-weight: `700`
- line-height: `1.2`

Sub-text styling:
- margin-top: `8px`
- font-size: `0.8rem`
- color: `rgba(253,253,253,0.4)`

### The 6 KPI Cards (in order):

| # | Label | Value (example) | Sub-text | Value color |
|---|---|---|---|---|
| 1 | EINNAHMEN (NETTO, JAHR) | € 120.000,00 | Summe Einnahmen-Transaktionen | `#6FC773` (green) |
| 2 | AUSGABEN (NETTO, JAHR) | € 28.000,00 | Summe Ausgaben-Transaktionen | `#E14A4D` (red) |
| 3 | GESCHÄTZTE UST. (EINNAHMEN) | € 27.600,00 | Netto × USt-% (ohne freigestellt) | `#FDFDFD` (white) |
| 4 | OFFEN: STEUERN & VERSICHERUNG (MODELL) | € 18.400,00 | Inkl. 50%-Einnahmen-Ansatz & SV/ESt.-Felder | `#FBDF62` (yellow) |
| 5 | INVESTITIONSBUDGET VERFÜGBAR | € 5.200,00 | Budget € 9.000,00 − gebucht € 3.800,00 | `#78B2FA` (blue) |
| 6 | BENÖTIGTES BANKGUTHABEN | € 42.000,00 | Liquiditätsplan (alle Posten) | `#C266E2` (purple), size `1.45rem` |

---

## 3. SECTIONS (detail data rows)

Each section follows this pattern:

```
[SECTION TITLE — UPPERCASE, bold, 1rem]
[Section description — muted, 0.85rem, max-width 720px]
[Data rows table — max-width 800px]
  Label (bold, width 180px)  |  Value (normal)
  Label                      |  Value
  ...
```

Section styling:
- margin-bottom: `32px`

Section title:
- font-size: `1rem`
- font-weight: `700`
- text-transform: UPPERCASE
- letter-spacing: `0.04em`
- color: `#FDFDFD`
- margin: `0 0 8px`

Section description:
- font-size: `0.85rem`
- color: `#8E8E8E`
- line-height: `1.45`
- max-width: `720px`
- margin: `0 0 14px`

Data row:
- `display: flex; gap: 10px`
- `.label`: `font-weight: 700; width: 180px`
- `.value`: flex: 1

Table max-width: `800px`

---

### SECTION 1: Buchhaltungsjahr 2026 — Konfiguration (ACF)

**Description:** Werte aus dem Buchhaltungsjahr-Post (gleiche Felder wie im Formular „Buchhaltungs Jahr bearbeiten").

| Label | Example Value |
|---|---|
| Jahr abgeschlossen | Nein |
| Offene Kosten | € 2.500,00 |
| Urlaubsgeld (Budget) | € 3.000,00 |
| Urlaub (Soll) | € 8.000,00 |
| Urlaub (genutzt) | € 2.000,00 |
| Krankenstand (Soll) | € 5.000,00 |
| Krankenstand (genutzt) | € 0,00 |
| Investitionsbudget / Monat | € 500,00 |
| Ausgabenbudget / Jahr | € 15.000,00 |
| Bestehender Emergency Fund | € 10.000,00 |
| Emergency Fund (Jahresziel) | € 12.000,00 |
| Vorauszahlungen | € 1.200,00 |
| ESt. bezahlt (laut Feld) | € 4.500,00 |
| Offener SV-Beitrag (laut Feld) | € 0,00 |
| USt. Q1 bezahlt | Ja |
| USt. Q2 bezahlt | Nein |
| USt. Q3 bezahlt | Nein |
| USt. Q4 bezahlt | Nein |

---

### SECTION 2: Einnahmen & Transaktionen

**Description:** Aus WordPress-Transaktionen; Jahr = Rechnungsdatum, sonst Buchungsdatum.

| Label | Example Value |
|---|---|
| Einnahmen netto (laufendes Kalenderjahr) | € 120.000,00 |
| Ausgaben netto (laufendes Kalenderjahr) | € 28.000,00 |
| Einnahmen netto (nur „offene" Buchhaltungsjahre) | € 120.000,00 |
| Ausgaben netto (nur „offene" Buchhaltungsjahre) | € 28.000,00 |
| Summe geschätzte USt. auf Einnahmen (Jahr) | € 27.600,00 |

---

### SECTION 3: Steuern, Versicherung & Rückstellungen

**Description:** „Versicherungs"-Ansatz = 50 % der Einnahmen in offenen Jahren; ESt.-Summe über alle noch nicht abgeschlossenen Jahre; SV offen für abgeschlossene Jahre.

| Label | Example Value |
|---|---|
| Rückstellung 50 % der Einnahmen (offene Jahre) | € 60.000,00 |
| Summe Ausgaben (offene Jahre, netto) | € 28.000,00 |
| ESt. bezahlt (Summe über offene Jahre, Konfig) | € 4.500,00 |
| Offener SV (Summe abgeschlossene Jahre, Konfig) | € 0,00 |
| ESt. bezahlt (Feld aktuelles Jahr) | € 4.500,00 |
| Offener SV-Beitrag (Feld aktuelles Jahr) | € 0,00 |
| Offene ESt. / Steuern / Versicherung (Kernformel) | € 27.500,00 |

---

### SECTION 4: Investitionen

**Description:** Monatsbudget × Monate im Jahr; Ausgaben „Investition" im laufenden Jahr.

| Label | Example Value |
|---|---|
| Investitionsbudget (kumuliert YTD) | € 1.500,00 |
| Investitionen gebucht (netto) | € 320,00 |
| Investitionen gebucht (brutto, Näherung) | € 393,60 |
| Verfügbar (Budget − netto gebucht) | € 1.180,00 |

---

### SECTION 5: Urlaub, Krankenstand & Puffer

**Description:** Hochrechnung pro Monat aus Konfig; Urlaubsgeld-Zusatz wie in der alten Formel (Modulo 6), falls Feld vacation_money gesetzt.

| Label | Example Value |
|---|---|
| Urlaub (Rest, hochgerechnet) | € 0,00 |
| Urlaubsgeld-Anteil (Modulo-6-Formel) | € 0,00 |
| Krankenstand (Rest, hochgerechnet) | € 1.250,00 |
| Summe Urlaub + Kranken + Urlaubsgeld | € 1.250,00 |
| Emergency Fund bestehend + anteilig Jahr | € 12.750,00 |
| — bestehend | € 10.000,00 |
| — anteilig laufendes Jahr | € 2.750,00 |
| Vorauszahlungen | € 1.200,00 |

---

### SECTION 6: Ausgabenbudget

**Description:** Laufendes Jahr: Ausgaben ohne Investitionsbuchungen und ohne den gewählten SV-Anteil der Ausgaben im Jahr.

| Label | Example Value |
|---|---|
| Ausgabenbudget (Jahr) | € 15.000,00 |
| Ausgaben (bereinigt) | € 11.280,00 |
| Saldo Budget | € 3.720,00 |

---

### SECTION 7: Liquidität gesamt

No description.

One **emphasis row** — label + large bold value:

| Label | Example Value |
|---|---|
| Benötigtes Guthaben auf dem Bankkonto | € 44.630,00 |

Emphasis value styling:
- font-size: `1.25rem`
- font-weight: `700`
- color: `#FDFDFD`

---

## 4. LOADING STATE

When data is still loading, instead of the full page, show:

```
<p class="accounting-stats-loading">Lade Kennzahlen…</p>
```

Styling: `color: #8E8E8E`

---

## 5. COMPLETE CSS (existing implementation)

Use these exact class names in your redesign if you want to stay compatible with the existing code.

```css
/* Root container */
.dls-accounting-stats {
  /* inherits page padding (20px) from main wrapper */
}

/* Page header */
.accounting-stats-header {
  margin-bottom: 28px;
}
.accounting-stats-header h2 {
  margin-bottom: 8px;
}
.accounting-stats-meta {
  margin: 0;
  font-size: 0.9rem;
  color: #8E8E8E;
}
.accounting-stats-warn {
  color: #FBDF62;
}
.accounting-stats-loading {
  margin: 0;
  color: #8E8E8E;
}

/* KPI grid */
.accounting-stats-kpi-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));
  gap: 16px;
  margin-bottom: 36px;
}

/* KPI card */
.accounting-stats-kpi {
  border: 1px solid #2A2A2A;
  border-radius: 8px;
  padding: 16px 18px;
  background: #1E1E1E;
}
.accounting-stats-kpi-label {
  font-size: 0.75rem;
  font-weight: 600;
  letter-spacing: 0.06em;
  text-transform: uppercase;
  color: #8E8E8E;
  margin-bottom: 10px;
  line-height: 1.35;
}
.accounting-stats-kpi-value {
  font-size: 1.35rem;
  font-weight: 700;
  color: #FDFDFD;
  line-height: 1.2;
}
.accounting-stats-kpi-sub {
  margin-top: 8px;
  font-size: 0.8rem;
  color: rgba(253,253,253,0.4);
  line-height: 1.35;
}

/* KPI color variants */
.accounting-stats-kpi.kpi-income .accounting-stats-kpi-value { color: #6FC773; }
.accounting-stats-kpi.kpi-expense .accounting-stats-kpi-value { color: #E14A4D; }
.accounting-stats-kpi.kpi-warning .accounting-stats-kpi-value { color: #FBDF62; }
.accounting-stats-kpi.kpi-invest .accounting-stats-kpi-value { color: #78B2FA; }
.accounting-stats-kpi.kpi-bank .accounting-stats-kpi-value {
  color: #C266E2;
  font-size: 1.45rem;
}

/* Sections */
.accounting-stats-section {
  margin-bottom: 32px;
}
.accounting-stats-section-title {
  font-size: 1rem;
  font-weight: 700;
  letter-spacing: 0.04em;
  text-transform: uppercase;
  color: #FDFDFD;
  margin: 0 0 8px;
}
.accounting-stats-section-desc {
  margin: 0 0 14px;
  font-size: 0.85rem;
  line-height: 1.45;
  color: #8E8E8E;
  max-width: 720px;
}
.accounting-stats-table {
  max-width: 800px;
}

/* Data rows (shared with .data-table) */
.data-table {
  display: flex;
  flex-direction: column;
  gap: 12px;
  padding: 10px 20px 20px;
}
.data-table-row {
  display: flex;
  gap: 10px;
}
.data-table-row .label {
  font-weight: 700;
  width: 180px;
}

/* Emphasis row (liquidity total) */
.accounting-stats-emphasis .value {
  font-size: 1.25rem;
  font-weight: 700;
  color: #FDFDFD;
}
```

---

## 6. COMPLETE HTML STRUCTURE (simplified, with example values)

```html
<div class="dls-accounting-stats">

  <!-- PAGE HEADER -->
  <div class="site-meta accounting-stats-header">
    <h2>Buchhaltung — Übersicht</h2>
    <p class="accounting-stats-meta">
      Kalenderjahr 2026, Monat 3 · Datenstand: 2026-03
    </p>
  </div>

  <!-- KPI GRID -->
  <div class="accounting-stats-kpi-grid">

    <div class="accounting-stats-kpi kpi-income">
      <div class="accounting-stats-kpi-label">Einnahmen (netto, Jahr)</div>
      <div class="accounting-stats-kpi-value">€ 120.000,00</div>
      <div class="accounting-stats-kpi-sub">Summe Einnahmen-Transaktionen</div>
    </div>

    <div class="accounting-stats-kpi kpi-expense">
      <div class="accounting-stats-kpi-label">Ausgaben (netto, Jahr)</div>
      <div class="accounting-stats-kpi-value">€ 28.000,00</div>
      <div class="accounting-stats-kpi-sub">Summe Ausgaben-Transaktionen</div>
    </div>

    <div class="accounting-stats-kpi">
      <div class="accounting-stats-kpi-label">Geschätzte USt. (Einnahmen)</div>
      <div class="accounting-stats-kpi-value">€ 27.600,00</div>
      <div class="accounting-stats-kpi-sub">Netto × USt-% (ohne freigestellt)</div>
    </div>

    <div class="accounting-stats-kpi kpi-warning">
      <div class="accounting-stats-kpi-label">Offen: Steuern &amp; Versicherung (Modell)</div>
      <div class="accounting-stats-kpi-value">€ 18.400,00</div>
      <div class="accounting-stats-kpi-sub">Inkl. 50%-Einnahmen-Ansatz &amp; SV/ESt.-Felder</div>
    </div>

    <div class="accounting-stats-kpi kpi-invest">
      <div class="accounting-stats-kpi-label">Investitionsbudget verfügbar</div>
      <div class="accounting-stats-kpi-value">€ 5.200,00</div>
      <div class="accounting-stats-kpi-sub">Budget € 9.000,00 − gebucht € 3.800,00</div>
    </div>

    <div class="accounting-stats-kpi kpi-bank">
      <div class="accounting-stats-kpi-label">Benötigtes Bankguthaben</div>
      <div class="accounting-stats-kpi-value">€ 44.630,00</div>
      <div class="accounting-stats-kpi-sub">Liquiditätsplan (alle Posten)</div>
    </div>

  </div>

  <!-- SECTION 1: Config -->
  <section class="accounting-stats-section">
    <h3 class="accounting-stats-section-title">Buchhaltungsjahr 2026 — Konfiguration (ACF)</h3>
    <p class="accounting-stats-section-desc">Werte aus dem Buchhaltungsjahr-Post (gleiche Felder wie im Formular „Buchhaltungs Jahr bearbeiten").</p>
    <div class="data-table accounting-stats-table">
      <div class="data-table-row"><div class="label">Jahr abgeschlossen</div><div class="value">Nein</div></div>
      <div class="data-table-row"><div class="label">Offene Kosten</div><div class="value">€ 2.500,00</div></div>
      <div class="data-table-row"><div class="label">Urlaubsgeld (Budget)</div><div class="value">€ 3.000,00</div></div>
      <div class="data-table-row"><div class="label">Urlaub (Soll)</div><div class="value">€ 8.000,00</div></div>
      <div class="data-table-row"><div class="label">Urlaub (genutzt)</div><div class="value">€ 2.000,00</div></div>
      <div class="data-table-row"><div class="label">Krankenstand (Soll)</div><div class="value">€ 5.000,00</div></div>
      <div class="data-table-row"><div class="label">Krankenstand (genutzt)</div><div class="value">€ 0,00</div></div>
      <div class="data-table-row"><div class="label">Investitionsbudget / Monat</div><div class="value">€ 500,00</div></div>
      <div class="data-table-row"><div class="label">Ausgabenbudget / Jahr</div><div class="value">€ 15.000,00</div></div>
      <div class="data-table-row"><div class="label">Bestehender Emergency Fund</div><div class="value">€ 10.000,00</div></div>
      <div class="data-table-row"><div class="label">Emergency Fund (Jahresziel)</div><div class="value">€ 12.000,00</div></div>
      <div class="data-table-row"><div class="label">Vorauszahlungen</div><div class="value">€ 1.200,00</div></div>
      <div class="data-table-row"><div class="label">ESt. bezahlt (laut Feld)</div><div class="value">€ 4.500,00</div></div>
      <div class="data-table-row"><div class="label">Offener SV-Beitrag (laut Feld)</div><div class="value">€ 0,00</div></div>
      <div class="data-table-row"><div class="label">USt. Q1 bezahlt</div><div class="value">Ja</div></div>
      <div class="data-table-row"><div class="label">USt. Q2 bezahlt</div><div class="value">Nein</div></div>
      <div class="data-table-row"><div class="label">USt. Q3 bezahlt</div><div class="value">Nein</div></div>
      <div class="data-table-row"><div class="label">USt. Q4 bezahlt</div><div class="value">Nein</div></div>
    </div>
  </section>

  <!-- SECTION 2: Income -->
  <section class="accounting-stats-section">
    <h3 class="accounting-stats-section-title">Einnahmen &amp; Transaktionen</h3>
    <p class="accounting-stats-section-desc">Aus WordPress-Transaktionen; Jahr = Rechnungsdatum, sonst Buchungsdatum.</p>
    <div class="data-table accounting-stats-table">
      <div class="data-table-row"><div class="label">Einnahmen netto (laufendes Kalenderjahr)</div><div class="value">€ 120.000,00</div></div>
      <div class="data-table-row"><div class="label">Ausgaben netto (laufendes Kalenderjahr)</div><div class="value">€ 28.000,00</div></div>
      <div class="data-table-row"><div class="label">Einnahmen netto (nur „offene" Buchhaltungsjahre)</div><div class="value">€ 120.000,00</div></div>
      <div class="data-table-row"><div class="label">Ausgaben netto (nur „offene" Buchhaltungsjahre)</div><div class="value">€ 28.000,00</div></div>
      <div class="data-table-row"><div class="label">Summe geschätzte USt. auf Einnahmen (Jahr)</div><div class="value">€ 27.600,00</div></div>
    </div>
  </section>

  <!-- SECTION 3: Taxes -->
  <section class="accounting-stats-section">
    <h3 class="accounting-stats-section-title">Steuern, Versicherung &amp; Rückstellungen</h3>
    <p class="accounting-stats-section-desc">„Versicherungs"-Ansatz = 50 % der Einnahmen in offenen Jahren; ESt.-Summe über alle noch nicht abgeschlossenen Jahre; SV offen für abgeschlossene Jahre.</p>
    <div class="data-table accounting-stats-table">
      <div class="data-table-row"><div class="label">Rückstellung 50 % der Einnahmen (offene Jahre)</div><div class="value">€ 60.000,00</div></div>
      <div class="data-table-row"><div class="label">Summe Ausgaben (offene Jahre, netto)</div><div class="value">€ 28.000,00</div></div>
      <div class="data-table-row"><div class="label">ESt. bezahlt (Summe über offene Jahre, Konfig)</div><div class="value">€ 4.500,00</div></div>
      <div class="data-table-row"><div class="label">Offener SV (Summe abgeschlossene Jahre, Konfig)</div><div class="value">€ 0,00</div></div>
      <div class="data-table-row"><div class="label">ESt. bezahlt (Feld aktuelles Jahr)</div><div class="value">€ 4.500,00</div></div>
      <div class="data-table-row"><div class="label">Offener SV-Beitrag (Feld aktuelles Jahr)</div><div class="value">€ 0,00</div></div>
      <div class="data-table-row"><div class="label">Offene ESt. / Steuern / Versicherung (Kernformel)</div><div class="value">€ 27.500,00</div></div>
    </div>
  </section>

  <!-- SECTION 4: Investments -->
  <section class="accounting-stats-section">
    <h3 class="accounting-stats-section-title">Investitionen</h3>
    <p class="accounting-stats-section-desc">Monatsbudget × Monate im Jahr; Ausgaben „Investition" im laufenden Jahr.</p>
    <div class="data-table accounting-stats-table">
      <div class="data-table-row"><div class="label">Investitionsbudget (kumuliert YTD)</div><div class="value">€ 1.500,00</div></div>
      <div class="data-table-row"><div class="label">Investitionen gebucht (netto)</div><div class="value">€ 320,00</div></div>
      <div class="data-table-row"><div class="label">Investitionen gebucht (brutto, Näherung)</div><div class="value">€ 393,60</div></div>
      <div class="data-table-row"><div class="label">Verfügbar (Budget − netto gebucht)</div><div class="value">€ 1.180,00</div></div>
    </div>
  </section>

  <!-- SECTION 5: Vacation / sick / buffer -->
  <section class="accounting-stats-section">
    <h3 class="accounting-stats-section-title">Urlaub, Krankenstand &amp; Puffer</h3>
    <p class="accounting-stats-section-desc">Hochrechnung pro Monat aus Konfig; Urlaubsgeld-Zusatz wie in der alten Formel (Modulo 6), falls Feld vacation_money gesetzt.</p>
    <div class="data-table accounting-stats-table">
      <div class="data-table-row"><div class="label">Urlaub (Rest, hochgerechnet)</div><div class="value">€ 0,00</div></div>
      <div class="data-table-row"><div class="label">Urlaubsgeld-Anteil (Modulo-6-Formel)</div><div class="value">€ 0,00</div></div>
      <div class="data-table-row"><div class="label">Krankenstand (Rest, hochgerechnet)</div><div class="value">€ 1.250,00</div></div>
      <div class="data-table-row"><div class="label">Summe Urlaub + Kranken + Urlaubsgeld</div><div class="value">€ 1.250,00</div></div>
      <div class="data-table-row"><div class="label">Emergency Fund bestehend + anteilig Jahr</div><div class="value">€ 12.750,00</div></div>
      <div class="data-table-row"><div class="label">— bestehend</div><div class="value">€ 10.000,00</div></div>
      <div class="data-table-row"><div class="label">— anteilig laufendes Jahr</div><div class="value">€ 2.750,00</div></div>
      <div class="data-table-row"><div class="label">Vorauszahlungen</div><div class="value">€ 1.200,00</div></div>
    </div>
  </section>

  <!-- SECTION 6: Expense budget -->
  <section class="accounting-stats-section">
    <h3 class="accounting-stats-section-title">Ausgabenbudget</h3>
    <p class="accounting-stats-section-desc">Laufendes Jahr: Ausgaben ohne Investitionsbuchungen und ohne den gewählten SV-Anteil der Ausgaben im Jahr.</p>
    <div class="data-table accounting-stats-table">
      <div class="data-table-row"><div class="label">Ausgabenbudget (Jahr)</div><div class="value">€ 15.000,00</div></div>
      <div class="data-table-row"><div class="label">Ausgaben (bereinigt)</div><div class="value">€ 11.280,00</div></div>
      <div class="data-table-row"><div class="label">Saldo Budget</div><div class="value">€ 3.720,00</div></div>
    </div>
  </section>

  <!-- SECTION 7: Liquidity total (emphasis) -->
  <section class="accounting-stats-section">
    <h3 class="accounting-stats-section-title">Liquidität gesamt</h3>
    <div class="data-table accounting-stats-table">
      <div class="data-table-row accounting-stats-emphasis">
        <div class="label">Benötigtes Guthaben auf dem Bankkonto</div>
        <div class="value">€ 44.630,00</div>
      </div>
    </div>
  </section>

</div>
```

---

## 7. DESIGN NOTES FOR REDESIGN

- **The KPI grid is the hero.** It should be immediately visible above the fold with no scrolling. Consider making it visually dominant — large cards, strong color, clear hierarchy.
- **The sections below are reference data.** They don't need to be as visually bold. A clean key-value table on a slightly elevated surface works well.
- **The 6 KPI cards always appear in this order:** income → expense → VAT estimate → tax/insurance reserve → investment budget → bank target. The last one (bank target, purple) is the most important number on the page.
- **Monetary values use European format:** `€ 44.630,00` (period as thousands separator, comma as decimal). Do not change this format.
- **The page is scrollable.** There are 7 sections below the KPI grid. They can be collapsed, tabbed, or shown in a different layout — but all data must be accessible.
- **No interactive elements** on this page currently — it is read-only. The data refreshes from the database. There are no buttons, no forms, no editing.
- **The warning state** (no config for current year) should be visually distinct but not alarming — the yellow text inline in the meta line is intentional.
- **Loading state:** "Lade Kennzahlen…" in muted text, no spinner needed (or a small inline one).
