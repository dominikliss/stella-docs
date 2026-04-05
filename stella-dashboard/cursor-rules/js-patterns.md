# Cursor rule export: `js-patterns.mdc`

_Source of agent rule in theme repo: `.cursor/rules/js-patterns.mdc`._

---

# JS / React Conventions

## Data Fetching — EntityService
Always pass `acf_format: 'standard'` (handled inside `EntityService`).

```js
const entityService = new EntityService();
const { records: transactions, isResolving } = entityService.getEntities('transaction');
```

Never store the fetched records in a separate `useState`. Use them directly or derive with `useMemo`.

## Derived State — useMemo over useEffect
Use `useMemo` for any value computed from fetched data. Never use `useEffect` + `setState` for derived/filtered lists.

```js
// ✅ CORRECT
const filtered = useMemo(() =>
  (transactions ?? []).filter(t => t.acf.type === 'Einnahme'),
  [transactions]
);

// ❌ WRONG — causes extra render cycle, stale state bugs
const [filtered, setFiltered] = useState([]);
useEffect(() => { setFiltered(transactions.filter(...)); }, [transactions]);
```

## Date Filtering
ACF date fields return `Y-m-d` strings. Compare as strings — no Date parsing needed.

```js
transactions.filter(t =>
  t.acf.transaction_date >= dateStart &&
  t.acf.transaction_date <= dateEnd
)
```

For accounting/chart logic use `getAccountingDate(t)` from `helpers.js` — it returns `invoice_date` with fallback to `transaction_date`. Always filter and group transactions with this helper, never raw `transaction_date`.

When you must convert a `Y-m-d` string to a `Date` object, use `parseDateLocal(str)` from `helpers.js`. Do **not** use `new Date(str)` — it parses as UTC midnight and shifts dates by one day in UTC+1/+2 timezones.

## Numeric ACF Fields
ACF `text` fields always return strings. Use `parseFloat()` before arithmetic.

```js
sumBy(transactions, t => parseFloat(t.acf.price_net) || 0)
```

## Custom REST API — apiFetch
Use `@wordpress/api-fetch` for browser-facing internal endpoints (handles nonce automatically).
Always use the `dls/v1` namespace — never `api/v1` (IP-restricted).

```js
import apiFetch from '@wordpress/api-fetch';
const data = await apiFetch({ path: `/dls/v1/your-endpoint?param=value` });
// DELETE with query string (e.g. revoke portal user):
await apiFetch({ path: `/dls/v1/client-portal-user?client_id=${id}`, method: 'DELETE' });
```

**Client portal:** `POST /dls/v1/client-portal-user` with `{ client_id }` creates the `customer` user; response may include `agb_file_created` / `agb_file_id` when an AGB template file was auto-created. **`DELETE /dls/v1/client-portal-user?client_id=`** removes only that portal user — not the client or files.

## Forms — Formik
All create/edit forms use Formik + Yup. For form side-effects (e.g. auto-fetching a rate when a field changes), use a `useFormikContext` child component — never put hooks inside the Formik render prop.

```js
function SideEffectWatcher() {
    const { values, setFieldValue } = useFormikContext();
    useEffect(() => { /* fetch and setFieldValue */ }, [values.someField]);
    return null;
}
// Inside <Formik>:
// <SideEffectWatcher />
```

**Conditional Yup:** use `.when('otherField', …)` for fields that only apply to a subset of `type` values (e.g. `document_type` when `type === 'document'` in `file-form.js`).

**Client form — new person in sidebar:** After creating a `person` from the client form sidebar, a small sync component (`SelectNewPersonInClientFormSync` pattern in `client-form.js`) should **`setFieldValue`** the new person into the Kontaktpersonen list and **`invalidateResolution`** for `person` so the new record appears in `SearchableSelect` options.

**Person form — birthday:** Use native **`<Field type="date" />`** (same pattern as `transaction-form.js`). Map ACF `d.m.Y` ↔ REST `Y-m-d` with helpers such as `acfDmYToIso` / inverse when loading and saving.

**Logo / Profilbild:** Hide the field **label** when an image is already selected (UX in `client-form.js` / `person-form.js`).

**Kontaktpersonen row layout:** Reference rows use **`field-array reference`** (`className="field-array reference"`). Keep icon, label, `SearchableSelect`, and **+/-** on **one flex row** — see `form.scss` (`.field-array.reference .field-array-row`). Width rules target `input:not(.searchable-select-input)` so searchable inputs can be `width: 100%`; **button margin** for reference rows may need a **more specific** rule after generic `.field-array .field-array-row .button` overrides.

## Shared Components
- **Delete modal** → `<DeleteModal label={...} onConfirm={...} onCancel={...} />` from `./delete-modal`. Never inline the modal HTML.
- **Toggle switch** → `<Toggle checked={bool} onChange={fn} label="…" />` from `./toggle`. Never use MUI `<Switch>`.
- **Dropdown/select** → `<Dropdown value={…} label="…" options={[{value, label}]} onChange={fn} />` from `./dropdown`. For a trigger-only button with a flyout menu (e.g. Export), pass `options` with `href` and omit `onChange`.
- **Searchable select** → `<SearchableSelect name="…" label="…" options={[{value, label}]} emptyValue="…" />` from `./searchable-select`. Use for long lists (clients, people, files, templates). **`emptyValue`** defaults to `''`; use **`emptyValue={0}`** when “no selection” is stored as numeric `0` (e.g. `tracking_time_pro_id`). The clear (×) control is hidden when `value === emptyValue`. Wire with Formik `<Field component={SearchableSelect} … />` or spread `field`/`meta` as in `client-form.js` / `file-form.js`.
- **Icons** → import named icons from `./icons` (`IconDownload`, `IconCreatePdf`, `IconSendEmail`, `IconWarning`, `IconEdit`, …). They use `fill="currentColor"` and default to `1em` — do not set explicit `width`/`height` props. Do not add inline `style` margins on icons inside `.data-table-row`.
- **Icons + `icons.scss`:** Global rules add a **font glyph** via `::before` on classes like `.icon-edit`. If you render **both** `<IconEdit className="icon-edit" />` and the SCSS `::before`, users see **double icons**. Prefer **SVG-only** in those spots: keep `className="icon-edit"` for sizing but **suppress** `::before` in scoped SCSS (e.g. `.field-array.reference .icon-edit::before { display: none; }`, file-form template picker). Do not remove shared `icons.scss` rules site-wide.
- **Sidebar form panel** → wrap forms in `<SidebarForm isOpen={bool} onClose={fn} title="…" wide className="…">…</SidebarForm>` from `./sidebar-form`. Use **`wide`** for TipTap / file form. Use **`className`** for stacking (e.g. `sidebar-form--template-edit` when opening a **nested** sidebar to edit a template from `file-form.js` — higher `z-index` in `form.scss`). Never build your own fixed-positioned panel.
- **Document lists** → `<DocumentList documents={…} emptyText="…" … />` from `./document-list` for per-client file rows and **empty states**. Use the same component + `emptyText` on **Reports** and **Dateien** so empty styling stays consistent (`document-list-empty`).
- **Expanding detail rows** → use `<ExpandingContent isOpen={bool}>…</ExpandingContent>` from `./expanding-content` for animated height expand/collapse inside table detail rows.
- **Loading spinner** → `<Loader />` from `./loader`. Inside a button: use inline alongside text (`<><Loader /> Saving…</>`).
- **Skeleton loading** → `<SkeletonRows rows={7} cols={5} actionCols={2} />` from `./skeleton` as a drop-in while `isResolving` is true.

## Toasts
Use the `useToast` hook (from `../hooks/use-toast`) for all user feedback. Always mount `<ToastContainer toasts={toasts} />` once at the top of the page component.

```js
import { useToast } from '../hooks/use-toast';
import ToastContainer from './toast-container';

const { toasts, addToast } = useToast();

// Success:
addToast('Saved successfully');
// Error:
addToast('Something went wrong', 'error');
```

Never use `alert()`, `console.log` displayed to the user, or custom inline error states for transient feedback.

## Currency & Number Formatting

```js
import { formatWithCurrency, getPriceInEur } from '../helpers';

formatWithCurrency(t.acf.price_net, t.acf.currency || 'EUR')
// → "€ 1,234.56" or "PLN 1,234.56"

getPriceInEur(t.acf.price_net, t.acf.currency, t.acf.pln_conversion_rate_invoice_date)
// → float in EUR (converts PLN → EUR using stored rate)
```

## Refreshing Entity Data After Mutations
After a successful `apiFetch` POST/PUT/DELETE that changes WordPress post data (e.g. import, bulk update, portal create/revoke), invalidate the entity cache so React re-fetches without a page reload:

```js
import { useDispatch } from '@wordpress/data';

const { invalidateResolution } = useDispatch('core');

// After the mutation succeeds:
invalidateResolution('getEntityRecords', [
  'postType',
  'transaction',
  { per_page: -1, order: 'asc', orderby: 'title', acf_format: 'standard' },
]);
```

The query object must match exactly what `EntityService.getEntities()` passes to `useEntityRecords`.

**Files / clients:** Reuse the **same** query args as in `EntityService` / `clients.js` (e.g. `FILE_QUERY` / `FILE_LIST_QUERY` in `file-form.js` for `postType` `file`). After portal user create, if an AGB file was added, invalidate **`file`** as well as **`client`** (so `dls_has_portal_user` and lists refresh).

## Conditional Badges / Labels
Use a `<span className="label-ksef">KSeF</span>` element (styled via `.label-ksef` in `styles.scss`) to indicate KSeF-imported transactions. Check the ACF field — **not** `t.meta`:

```jsx
{t.acf?.ksef_reference && (
  <span className="label-ksef">KSeF</span>
)}
```

## Imports
- WordPress hooks: `import { useState, useMemo, useEffect } from '@wordpress/element'`
- WordPress data: `import { useDispatch } from '@wordpress/data'` (for `invalidateResolution`)
- Only import hooks that are actually used — remove unused `useEffect` imports.
- `ReactDOM` is only needed in the file that mounts to a DOM container (bottom of page-level components).
