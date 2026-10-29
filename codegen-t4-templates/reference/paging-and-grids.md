# Paging and grid rules

Read this when: you change paging, search, sorting or the grid columns of a generated screen in any stack.

## Paging rules (every stack that has the screen)

- Limits in the endpoint: `Math.Clamp(pageSize, 1, 1000)` first (the answer carries the size used); pageNumber below 1 or an overflowing offset is a `Results.Problem` 400, never `BadRequest("text")`. `API_Crud` `GetAll` pages when `pageNumber`/`pageSize` is given and stays an array otherwise.
- Read the page again after a delete and **step back to the last page** when the page asked for is past the end (`if (items.length === 0 && totalCount > 0 && pageIndex > 0) { pageIndex = ceil(totalCount / pageSize) - 1; load(); return; }`); use the paginator's page size; say "Nothing found." only after the first response; generated specs test the first request and the step back.
- Why: a local filter after a delete leaves the total, the rows that move up and the last page wrong. "Nothing found." before the first response is wrong because the list is empty until then, so keep a `loaded` flag.
- Fix it in Angular (`TS_Component`, `TS_DetailMasterComponent`), React (`TSX_Page`, `TSX_DetailMasterPage`), Blazor (`BLZ_Page`) and WinUI3 (`WinUI3_MasterScreen` + `PaginationBar`).
- Compare the generated version with the hand-built one on real data volume before porting; measure and remove machinery that shows no benefit.
- `ListProtocol=Search` changes only the Angular client (one search box, `?pageIndex&pageSize&sort=col:dir&search=`, answer `{ data, count }`); the generated API still answers the Columns protocol. `DefaultSorts` gives a paged screen its starting sort.

## Page-template features across stacks

- **Page-template features across stacks** (grid columns, OData `$select`, owner scope): see the `odata-generated-grids` skill. A rule three stacks must share goes in Core with a same-lists test; helper code goes in a `DatabaseOnly` plan template, never in an essentials file.

## Opening a row of another screen and coming back (React and Angular)

A master-detail grid opens a child row on its own screen and returns to the parent's open dialog.

- URL: `/<child-kebab>?edit=<id>&back=<encoded parent url>`, for example `/sales-invoice?edit=31&back=%2Fcustomer-monthly-summary%3Fedit%3D1`.
- Each screen reads the query once on start: `edit` calls `getById` and then the screen's `edit(row)`; `back` navigates there after a successful save or Cancel.
- Route paths are the kebab-case of the table names and equal the hand-written routes; the API route plural follows the C# base class's plural rule (`Summary` to `summaries`, `ItemStatus` stays).
- The child grid shows the child table's own columns only: the API returns navigation properties as keys (`customer: null`), which would become empty columns.

## Client-side sort

- A client-side sort of a grid that holds all its rows compares by value (numbers as numbers, dates as dates, text without case, nothing first), a column that shows a name sorts by the name, a WinUI3 row carries raw `Values`; key the stored sort `<grid>.<child>`.
