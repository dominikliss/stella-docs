# Cursor rule export: `agent-workflow.mdc`

_Source of agent rule in theme repo: `.cursor/rules/agent-workflow.mdc`._

---

# Agent Workflow — How to Work in This Codebase

## 0. Plan before you code

**Every non-trivial task starts with a plan.** Before writing any code, outline what you're going to do:

1. **List the files** you expect to touch (read them first — see rule 1)
2. **Identify the steps** in order, with dependencies between them
3. **Use the TODO tool** (`TodoWrite`) for any task with 3+ steps — track each step's status as you go

For **complex or multi-file tasks** (new features, refactors touching 4+ files, unfamiliar areas of the codebase), **use subagents** to explore in parallel before starting implementation:

- Use `subagent_type="explore"` to search the codebase for patterns, find related files, or answer "how does X work?" questions
- Launch **multiple explore subagents in parallel** when you need to understand several areas at once (e.g. one for the PHP route, one for the React component, one for the SCSS)
- Use `subagent_type="shell"` for build/test verification that can run alongside other work
- Only start coding **after** your exploration subagents return and you understand the full picture

**Don't skip planning for speed.** A 30-second plan prevents 10 minutes of backtracking and broken code. When you catch yourself about to "just start writing" — stop, read, plan, then write.

## 1. Read before you edit

**Never edit a file you haven't read.** Always read the target file (and related files) first to understand context, existing patterns, and imports. Skipping this causes duplicated code, broken patterns, and regressions.

When adding to an existing component: read the full component to understand its state, props, and data flow before inserting new code.

## 2. Find the nearest existing pattern

Before building anything new, **find the most similar existing feature** and follow its pattern exactly. Examples:

- New CPT list page → study `invoices.js` or `products.js` (PageLayout + EntityListTable/EntityCardList + useEntityCrud)
- New form → study `transaction-form.js` or `product-form.js` (Formik + Yup + useFormSubmit)
- New REST route → study an existing `inc/routes/*.php` file (permission callback, response shape)
- New SCSS partial → study an existing `assets/scss/components/*.scss` file (import variables, BEM naming)

**Grep the codebase** for similar patterns before writing new code. If `useEntityCrud` already handles sidebar + delete modal + toast + cache invalidation, don't re-implement those manually.

## 3. Build after JS changes

After any change to `src/**/*.js`, verify the build compiles:

```bash
npx wp-scripts build
```

A successful build exits 0 with only size warnings. Fix any errors before declaring the task done.

## 4. Check lints after edits

Use `ReadLints` on every file you edited. Fix any linter errors you introduced. Don't fix pre-existing lints unless they're in code you touched.

## 5. User-visible strings stay German

All UI text (labels, toasts, headings, placeholders, modal descriptions, empty states) must be in **German**. Code identifiers (variables, functions, components, file names) must be in **English** (see `english-code-naming.mdc`). For artificial intelligence, product copy uses **„AI“**, not **„KI“** (e.g. „AI-Chat“, „AI-Anbindungen“, „E-Mail-AI-Analysen“).

```js
// ✅ Correct
addToast("Transaktion erstellt");
<PageLayout title="Buchhaltung">

// ❌ Wrong
addToast("Transaction created");
<PageLayout title="Accounting">
```

## 6. No narrating comments

Don't add comments that describe what the code does. Comments should only explain **why** something non-obvious is done.

```js
// ❌ Bad — narrates the obvious
// Import the page layout component
import PageLayout from "./page-layout";
// Set loading to true
setLoading(true);

// ✅ Good — explains non-obvious intent
// ACF returns date as d.m.Y but REST expects Y-m-d
const isoDate = acfDmYToIso(entity.acf.accepted_date);
```

## 7. Don't invent — ask

If you need something that doesn't exist (a new component, a new SCSS class, a new hook), **describe what you need and ask** before building it. Often the need is already covered by an existing abstraction listed in `reuse-components.mdc`.

## 8. Clean imports

- WordPress hooks: `import { useState, useMemo } from '@wordpress/element'` — not from `react`
- Only import what you use — remove unused imports
- `ReactDOM` only in files that mount to the DOM (bottom of page-level components)
- `import apiFetch from '@wordpress/api-fetch'` for REST calls — never `fetch()`

## 9. One concern at a time

When making changes, don't refactor unrelated code. Stay focused on the task. If you notice something broken nearby, mention it but don't fix it unless asked.

## 10. Documentation — `stella-docs` submodule (SSoT)

**Full policy:** `documentation-source-of-truth.mdc` (always applied).

Before finishing any task that **changes behaviour**, APIs, options, cron, DB schema, mail/embed/Stella integration, or ACF semantics:

1. Identify which file(s) under **`docs/stella-docs/`** need edits (`stella-dashboard/`, `stella-server/`, `integration/`, or `open-gaps.md`).
2. Update them **in the same task**; then **commit the submodule** and **bump the gitlink** in the theme repo (`git add docs/stella-docs`).

Skipping doc updates when the user-facing or operator-facing story changed is **not** complete. Trivial-only refactors (rename local variable, same HTTP contract) may omit docs—when in doubt, update.
