---
name: react-crud-gotchas
description: Concrete React (Vite, react-router, plain CSS) bugs and fixes found in a generated CRUD screen app over an ASP.NET API — a form opening off-screen under a long grid (use a native modal dialog), required fields on a hidden tab blocking Save silently, a nullable date that cannot be cleared, tab panels that lose their stacked layout, opening a row of another page and returning, search/paging parity between plain and master-detail pages, route-name contracts with the API, and how to check a page in a browser. Use when writing, generating or reviewing React pages that talk to an ASP.NET API, or when a generated page looks or behaves wrong.
---

# React CRUD screens: bugs to look for

## Forms

- **Form under a long grid**: the person clicks Edit on row 13 and nothing seems to happen (the form is below the fold). Render the form as a native modal dialog:
  `<dialog ref={(el) => { if (el && !el.open) el.showModal(); }} onCancel={(e) => { e.preventDefault(); cancel(); }}>`; style `dialog` (`width: min(920px, 94vw); max-height: 90vh; overflow: auto`) and `dialog::backdrop`. An error line that lives on the page is behind the backdrop: show it inside the dialog and hide the page-level one while it is open.
- **Required field on a tab that is not showing** (`hidden` panel): the browser blocks submit with no visible message. On the Submit button's `onClick`, find `form.querySelector(':invalid')?.closest('[role="tabpanel"]')` and `setTab` to its `data-tab`; React flushes that update before the native validation bubble.
- **Nullable date**: `<input type="date">` has an easy-to-miss clear. Put a **Clear** button beside it that sets the property to `undefined` (JSON drops it, the API stores NULL); bind `value={row.x ?? ''}` and `onChange` to `event.target.value || undefined`. Verified by reading the row back from the database.
- A decimal/percentage column wants `min`/`max` (a percentage 0-100; `decimal(5,2)` +-999.99) as well as `step`.
- Boolean checkboxes and labels in a tab panel stay on one line unless the panel is also a flex column (`form, [role='tabpanel'] { display: flex; flex-direction: column; }`) and `[role='tabpanel'][hidden] { display: none; }` is restored, because `display: flex` overrides the `hidden` attribute.

## Pages

- A master-detail page must have the same search bar and paging as a plain page (a Search box per searchable column, `getPage(page, size, filters)`, `PaginationBar`); the first version loaded everything with `getAll()`.
- **Child grid row -> its own page and back**: `navigate('/sales-invoice?edit=' + id + '&back=' + encodeURIComponent('/customer-monthly-summary?edit=' + parentId))`; every page reads `useSearchParams()` once (`edit` -> `api.getById` -> `edit(row)`; `back` -> `navigate(back)` after save or cancel). Pages that use router hooks need `<MemoryRouter>` in their tests.
- The API returns navigation properties as keys (`customer: null`); a grid that shows "whatever keys the row has" must keep only the child's real columns.
- Route names are a contract: `/api/<plural>` built by the generator's pluralizer must equal the C# base class's plural rule (`Summary` -> `summaries`, `ItemStatus` stays), and page paths are kebab-case of the table name.
- Menu with a hamburger: `useState(() => localStorage.getItem('menuOpen') !== 'false')`, wrap storage in try/catch, `.app.menu-closed { grid-template-columns: 1fr }`.

## Checking a page

`npm run build` (tsc + vite) and `npm test`; then load the page in the built-in browser and measure with `getBoundingClientRect` (see the `verify-in-running-app` skill). The API uses whatever database its connection string names: say so before the person clicks Delete.
