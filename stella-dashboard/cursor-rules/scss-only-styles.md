# Cursor rule export: `scss-only-styles.mdc`

_Source of agent rule in theme repo: `.cursor/rules/scss-only-styles.mdc`._

---

# Styles: SCSS source only

**Do not edit files under `assets/css/`** (`styles.css`, `login.css`, or other compiled outputs). They are **generated** from SCSS (ScssPhp on theme load in `functions.php` → `stella_compile_all_new_scss()`). Manual CSS edits are **lost** on the next compile.

## Do this instead

- Add or change rules in **`assets/scss/`** (partials under `assets/scss/components/`, `styles.scss`, `login.scss`, etc.).
- Use **`@import "variables"`** (or the existing import pattern) so tokens match the design system.
- After SCSS changes, rely on **WordPress recompiling** when SCSS is newer than `stella_last_scss_compile_timestamp`, or trigger a recompile by touching/saving SCSS as in normal theme workflow.

## Exceptions (rare)

- If the user explicitly asks to patch compiled CSS for a one-off hotfix **and** acknowledges it will be overwritten — still prefer SCSS unless they insist.
- Third-party vendor CSS that is not produced by this theme’s SCSS pipeline (if any) follows whatever the project already does.

When suggesting or applying style changes, **only touch `.scss` files** in this theme.
