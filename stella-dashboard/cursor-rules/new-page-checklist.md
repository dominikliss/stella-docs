# Cursor rule export: `new-page-checklist.mdc`

_Source of agent rule in theme repo: `.cursor/rules/new-page-checklist.mdc`._

---

# Adding a New Page or View

Follow this checklist when creating a new page-level component or splitting an existing view into its own route/tab.

## Page-level component pattern

Every page-level component should use `PageLayout` for structural consistency:

```jsx
import PageLayout from "./page-layout";
import ToastContainer from "./toast-container";
import { useToast } from "../hooks/use-toast";

export default function MyPage() {
  const { toasts, addToast } = useToast();

  return (
    <>
      <ToastContainer toasts={toasts} />
      <PageLayout
        tabs={[{ label: "Tab A", href: "/path/a", active: true }]}
        title="Seitentitel"
        actions={<button className="button button-purple">Aktion</button>}
      >
        {/* page body */}
      </PageLayout>
    </>
  );
}
```

### PageLayout tab props

- **Link tabs** (server navigation): `{ label, href, active }`
- **Button tabs** (client-side state): `{ label, onClick, active, icon? }`

When tabs switch client-side state (not full page loads), use `onClick`. When tabs navigate to different URLs, use `href`.

## CPT list page (standard pattern)

For pages that list/create/edit a WordPress custom post type, wire the standard hooks:

1. `useEntityCrud(postType, LIST_QUERY, 'Label')` — sidebar, delete modal, active row, toast
2. `EntityListTable` or `EntityCardList` for the list
3. `SidebarForm` + dedicated `*Form` component for create/edit
4. `DeleteModal` for delete confirmation

Study `invoices.js` (table-based) or `products.js` (card-based) as reference implementations.

## Routing checklist

When the page needs its own URL:

- [ ] Add route to `dls_get_app_routes()` in `inc/app-routes.php`
- [ ] Add `add_rewrite_rule` if not a simple top-level slug
- [ ] Add `<div class="dls-{slug}"></div>` mount point in the template render block
- [ ] Add `ReactDOM.render(…)` at the bottom of the component file targeting `.dls-{slug}`
- [ ] Flush permalinks (save Settings → Permalinks in WP admin)
- [ ] If it's a Verwaltung sub-tab: add to `dls_verwaltung_subroutes()` (and PHP `$tabs` in `app-routes.php`) and `admin-settings-page.js` `VALID_TABS` + render branch

## SCSS for new pages

- [ ] Create `assets/scss/components/{page-name}.scss`
- [ ] Start with `@import "../variables";`
- [ ] Import the new partial in the main `assets/scss/styles.scss`
- [ ] Use existing SCSS classes and tokens (see `reuse-components.mdc` and `design-system.mdc`)
- [ ] Never set `font-size` below `1rem` (see `min-font-size.mdc`)
