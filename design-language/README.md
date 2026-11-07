# Technical Precision — internal tools design language

Shared visual language for **new internal ops tools** (Atlas and anything that should feel like Atlas).

**Not** the ddashboard / WordPress theme language. That stays in [`../stella-dashboard/DESIGN_SYSTEM.md`](../stella-dashboard/DESIGN_SYSTEM.md) (purple accent, Museo Sans / Nexa, German, desktop-only). Do not mix the two.

| File | Role |
|------|------|
| [SPEC.md](SPEC.md) | The language: colour, type, layout, components, shell recipe |
| [tokens.json](tokens.json) | Machine-readable tokens. Copy these names into Tailwind / CSS |
| [auth-landing.md](auth-landing.md) | New Atlas **landing language** (pink chrome, glow, 8–12px radius, soft field focus) + WCAG 2.1 AA. Signed-in chrome still follows SPEC. |

**Reference implementation:** Atlas SPA (`resources/js`, `resources/css/app.css`, `tailwind.config.ts`). Atlas mockups live in the Atlas repo under `design/`.

---

## Who this is for

Any repo that vendors this submodule as `docs/stella-docs` and is building an internal tool (not ddashboard, not a marketing site).

```text
docs/stella-docs/design-language/SPEC.md
docs/stella-docs/design-language/tokens.json
```

Agents and humans in those repos treat this folder the same way Atlas Connect treats [`../atlas/atlas-connect/backups.md`](../atlas/atlas-connect/backups.md): implement from here, do not invent a second scale.

---

## How to use it in a new tool

1. Read [SPEC.md](SPEC.md).
2. Copy [tokens.json](tokens.json) into the app theme (Tailwind `@theme` / `theme.extend`, or CSS custom properties). Keep the token names (`layer-0`, `tech-border`, `h1`, `body-sm`, `status-success`, …).
3. Load **Inter**, **JetBrains Mono**, and **Material Symbols Outlined**.
4. Build the shell from SPEC §8 (220px sidebar, 48px top bar, `PageHeader`, dense tables).
5. Change only the product name, logo, and nav items.
6. Do not invent a second type scale, radius, accent, or light theme.

This is **spec + tokens**, not an npm package. A shared component library can come later. Until then, coherence comes from this folder.

---

## Atlas

Atlas **signed-in** chrome implements this SPEC. Unauthenticated screens (`/login`, `/forgot-password`, `/reset-password`) use the landing language in [auth-landing.md](auth-landing.md) — start there if the rest of the app is being redesigned to match login. See [`../atlas/README.md`](../atlas/README.md).
