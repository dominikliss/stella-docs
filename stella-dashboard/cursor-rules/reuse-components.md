# Cursor rule export: `reuse-components.mdc`

_Source of agent rule in theme repo: `.cursor/rules/reuse-components.mdc`._

---

# Reuse Generic Components and CSS — No One-Off Implementations

Before writing any new JSX structure or SCSS rule, **check if an existing component or class already covers the need**. If nothing fits, **ask the user** before building a new bespoke implementation.

## React components — always prefer these

| Need | Use |
|---|---|
| Slide-in form panel | `SidebarForm` (`sidebar-form.js`) |
| Confirm / destructive action | `DeleteModal` (`delete-modal.js`) with `description` prop for custom body |
| Loading spinner | `Loader` (`loader.js`) |
| Table skeleton rows | `SkeletonRows` (`skeleton.js`) |
| Searchable dropdown | `SearchableSelect` / `SearchableSelectStandalone` (`searchable-select.js`) |
| Custom toggle | `Toggle` (`toggle.js`) |
| Dropdown menu | `Dropdown` (`dropdown.js`) |
| Success / error feedback | `addToast` from `useToast` + `ToastContainer` (`toast-container.js`) |
| Email send dialog | `EmailModal` (`email-modal.js`) |
| Paginated table controls | `TablePagination` (`table-pagination.js`) |
| Debounced search field | `ListSearchField` (`list-search-field.js`) |
| Two-column master/detail layout | `MasterDetailLayout` (`master-detail-layout.js`) |
| Scrollable flex panel | `ScrollPanel` (`scroll-panel.js`) |
| Integration card with icon/badge | `IntegrationServiceCard` (`management-integration-card.js`) |
| SVG icons | `icons.js` — check there before creating inline SVG |
| Expanding/collapsing content | `ExpandingContent` (`expanding-content.js`) |
| Panel with header / loading / empty state | `EntityListPanel` (`entity-list-panel.js`) |
| Date picker in Formik form | `FormikDatePicker` (`formik-date-picker.js`) — stores `Y-m-d` (or `""`) |
| Chat bubble (left/right aligned) | `ChatBubble` (`chat-bubble.js`) — `align="start"` or `"end"` |
| Page content shell (tabs → heading → body) | `PageLayout` (`page-layout.js`) — optional `tabs`, `title`, `actions`, `filter`, `afterHeading`, `children` |
| REST mutation with loading | `useRestAction` (`use-rest-action.js`) — wraps apiFetch with loading state |
| Entity list CRUD lifecycle | `useEntityCrud` (`use-entity-crud.js`) — sidebar, delete modal, active row, toast, cache invalidation for CPT list pages |
| Form submit with toast | `useFormSubmit` (`use-form-submit.js`) — entity load, create/update branching, toast, error formatting for Formik forms |
| Custom REST data fetching | `useDlsQuery` (`use-dls-query.js`) — lightweight fetch hook with loading/error/refetch for `dls/v1` endpoints |
| Card-style entity list | `EntityCardList` (`entity-card-list.js`) — `.card > .data-list` with header, skeleton, edit/delete per item |
| Key-value detail table | `DataTable` + `DataTableRow` (`data-table.js`) — `.data-table` > `.data-table-row` > `.label` + `.value` |
| Table with expand rows | `EntityListTable` (`entity-list-table.js`) — `.data-grid` table with skeleton, empty state, `ExpandingContent` detail rows |
| Icon action button group | `CrudActions` (`crud-actions.js`) — flex row of `.button-crud` icon buttons/links |

## SCSS classes — always reuse

| Need | Class(es) |
|---|---|
| Dark card / panel box | `.card` (4px radius, `$black-4` bg) or `.dls-mail-accounts-panel` (8px radius, `$black-2` bg, `overflow:hidden`) or `.entity-panel` (8px radius, `$black-2` bg, `border: 1px solid $black-3`) |
| Page section with title + content gap | `section.management-integration-group` → `__intro` + content |
| Section title | `.management-integration-group__title` |
| Section subtitle / description | `.management-integration-group__lede` |
| Integration card shell | `.integration-service-card` |
| Form field row | `.field-row` > `.field-container` |
| Validation error | `.validation-error` |
| Buttons | `.button`, `.button-purple`, `.button-dark`, `.button-small`, `.button-crud` |
| Searchable dropdown | `.searchable-select` |
| Sidebar/panel form | `.sidebar-form` |

## Panel head pattern

When a panel needs a titled header row (title left, action right) **with built-in loading and empty states**, use `EntityListPanel` (`entity-list-panel.js`) — it renders `.entity-panel` with `__head`, `__title`, `__loader`, and `__empty` elements automatically.

For a bare card that needs only the header row (no loading/empty logic), reuse `.dls-mail-accounts-panel__head` + `.dls-mail-accounts-panel__title` rather than inventing a new header block.

## Rules

1. **Check first** — search for an existing component or class before writing a new one.
2. **One-off class = ask** — if no existing implementation fits and you would create a class used only once, stop and ask the user whether to extend an existing component, create a new reusable one, or proceed with a scoped override.
3. **No duplicate structure** — do not recreate styling in a new class that already exists (e.g. custom `border-radius`, `background`, `padding` that duplicates `.card` or `.dls-mail-accounts-panel`).
4. **SCSS only** — never edit `assets/css/` directly; changes belong in `assets/scss/` (see `scss-only-styles.mdc`).
