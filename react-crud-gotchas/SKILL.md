---
name: react-crud-gotchas
description: Use when writing, generating or reviewing React (Vite, react-router, plain CSS) pages that talk to an ASP.NET API, or when a generated CRUD page looks or behaves wrong. Trigger words - form opens off-screen, Save does nothing, required field on a hidden tab, date cannot be cleared, tab panel layout lost, child row and back, master-detail paging, page shows stale count after delete, route name mismatch.
---

# React CRUD screens: bugs to look for

Each table row is a mistake that was made: what was written, what to write, and why it fails.

## Forms

| Wrong | Right | Why |
|---|---|---|
| The edit form renders under a long grid | A native modal dialog: `<dialog ref={(el) => { if (el && !el.open) el.showModal(); }} onCancel={(e) => { e.preventDefault(); cancel(); }}>`; style `dialog` (`width: min(920px, 94vw); max-height: 90vh; overflow: auto`) and `dialog::backdrop`. Show an error line inside the dialog and hide the page-level one while it is open | The person clicks Edit on row 13 and nothing seems to happen: the form is below the fold. An error line on the page is behind the backdrop |
| A required field on a tab that is not showing (`hidden` panel) | On the Submit button's `onClick`, find `form.querySelector(':invalid')?.closest('[role="tabpanel"]')` and `setTab` to its `data-tab` | The browser blocks the submit with no visible message. React flushes that update before the native validation bubble |
| A nullable `<input type="date">` with no way to empty it | A **Clear** button beside it that sets the property to `undefined`; bind `value={row.x ?? ''}` and `onChange` to `event.target.value \|\| undefined` | The native clear is easy to miss. JSON drops `undefined`, so the API stores NULL. Verified by reading the row back from the database |
| A decimal or percentage input with only `step` | Add `min`/`max` as well (a percentage 0-100; `decimal(5,2)` +-999.99) | `step` does not bound the value |
| Boolean checkboxes and labels in a tab panel that sit on one line | `form, [role='tabpanel'] { display: flex; flex-direction: column; }` and restore `[role='tabpanel'][hidden] { display: none; }` | `display: flex` overrides the `hidden` attribute, so the second rule is needed once the first is added |

## Pages

| Wrong | Right | Why |
|---|---|---|
| A master-detail page that loads everything with `getAll()` | The same search bar and paging as a plain page: a Search box per searchable column, `getPage(page, size, filters)`, `PaginationBar` | The first version did this and had no search or paging |
| A child grid row that has no way to open its own page and return | `navigate('/sales-invoice?edit=' + id + '&back=' + encodeURIComponent('/customer-monthly-summary?edit=' + parentId))`; read `useSearchParams()` once on every page | The person must land back on the parent's open dialog (contract: `paging-and-grids` (codegen-t4-templates `reference/`)). Pages that use router hooks need `<MemoryRouter>` in their tests |
| A grid that shows "whatever keys the row has" | Keep only the child table's real columns | Navigation properties arrive as keys (`customer: null`); see `paging-and-grids` (codegen-t4-templates `reference/`) |
| An `/api/<plural>` route or page path guessed per page | The generator's pluralizer must equal the C# base class's plural rule; page paths are kebab-case of the table name (`paging-and-grids` (codegen-t4-templates `reference/`)) | Route names are a contract with the API |
| A grid or form that adds or edits a row and shows `row.employee?.name` | Set the related object after the save (`row.employee = list.find(...)`) | The API returns the row without navigation properties, so the column stays blank until a reload |

- Required marks (red `*`, marked control, Save stays enabled) and the control to use per field type (check box, radio buttons, drop-down, number box): the `ui-conventions` skill, sections 7 and 12.
- Menu with a hamburger: `useState(() => localStorage.getItem('menuOpen') !== 'false')`, wrap storage in try/catch, `.app.menu-closed { grid-template-columns: 1fr }`.

## Checking a page

`npm run build` (tsc + vite) and `npm test`; then load the page in the built-in browser and measure with `getBoundingClientRect` (see the `verify-in-running-app` skill). The API uses whatever database its connection string names: say so before the person clicks Delete.

## Server-paged pages

The shared rules are in `paging-and-grids` (codegen-t4-templates `reference/`).

| Wrong | Right | Why |
|---|---|---|
| Filtering the list in state after a delete | Read the page again with `load(page)` and step back to the last page when it comes back empty with `totalCount > 0`: `if (result.items.length === 0 && result.totalCount > 0 && targetPage > 1) { load(result.totalPages, ...); return; }` | The total pages and the last page go stale |
| `setPageSize(size); load(1, filters, sort)` | `setPageSize(size); load(1, filters, sort, size)`. Give `load` the size as its **last** parameter with the state as default, so every existing call keeps working | The state is not updated yet when `load` runs |
| A new prop on a shared component that a generator writes once (`PaginationBar.tsx`) | Put the page-size `<select>` and the "Nothing found." paragraph in the page itself | `PaginationBar.tsx` is an essentials file: a project made earlier keeps the old one, and a new required or unknown prop fails `tsc` |

- "Nothing found." only after a `loaded` flag is true (set after the first response).
