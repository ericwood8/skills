---
name: angular-crud-gotchas
description: Use when writing, generating or reviewing Angular components, services or specs (standalone components, Material) that talk to an ASP.NET API, or when ng test or ng build results look wrong. Trigger words - boolean select posts "true", date box, YYYY week-year, service.delete does nothing, NullInjectorError, NG05105, blank page after request succeeds, OnPush, mat-paginator, sort header, JSON casing, form off-screen.
---

# Angular CRUD screens: bugs to look for

Each table row is a mistake that was made: what was written, what to write, and why it fails.

## Forms and data

| Wrong | Right | Why |
|---|---|---|
| `<option value="true">` in a boolean `<select>` | `<option [ngValue]="true">` / `[ngValue]="false"`, or a checkbox | `value="true"` binds the **string** `"true"` (and shows blank for a real boolean). The API's bool binding rejects `"true"` strings |
| A date property typed as a date object, or `<input type="date">` bound to the API's raw value | Type date properties as `string`; trim in `edit()`: `row.date = row.date.substring(0, 10)`; display with `\| date:'MM/dd/yyyy'` | JSON has no date type: the API sends and accepts `"2025-12-25T00:00:00"`, and `<input type="date">` needs `yyyy-MM-dd` |
| `DatePipe.transform(x, 'YYYY-MM-dd')` | `yyyy` | `YYYY` is the *week-based* year: wrong in late December and early January |
| Editing the grid's row object directly | Edit a **copy** (`{ ...row }`) | Mutating the row changes what the grid shows |
| Sample defaults hard-coded in `add()` | `''`, `0`, today, or the database default | A sample value is not the database's default |
| A property named differently from the server's JSON (`SY_RequestStatusTypeId`) | Match the server's casing exactly (`sY_RequestStatusTypeId`, `e_TimeSheetId`) | A wrongly cased interface property reads `undefined` |
| `selectedRow?.id` inside `*ngIf="selectedRow"` | `selectedRow.id` | `?.` on a value `*ngIf` already narrowed gives NG8107 |
| An error handler that only `console.log`s | 400 = the API rejected the value (special characters; delete of a row in use), 404 = gone, else show `error.message` | The user gets no feedback |
| A grid or form that adds or edits a row and shows `row.employee?.name` | Set the related object after the save (`row.employee = list.find(...)`) | The API returns the row without navigation properties, so the column stays blank until a reload |

- Required marks (red `*`, marked control, Save stays enabled) and the control to use per field type (check box, radio buttons, drop-down, number box): the `ui-conventions` skill, sections 7 and 12.

## RxJS

| Wrong | Right | Why |
|---|---|---|
| `service.delete(id)` with no `.subscribe(...)` | Subscribe | HTTP observables are **cold**: nothing is sent |
| `subscribe(nextFn, errorFn)` | `subscribe({ next: ..., error: ... })` | The two-callback form is deprecated. To rewrite many: find each `.subscribe(` with two top-level arguments, balance parentheses, skip string literals and `//` comments, and re-indent the bodies by two spaces |
| Two independent loads that must combine (sheets need the employee list to show names) | Chain them | They race |

## Tests and builds

| Wrong | Right | Why |
|---|---|---|
| The CLI's stock `should create` spec for a component whose service uses `HttpClient` | `providers: [provideHttpClient(), provideHttpClientTesting()]`; with Material or `RouterLink` also `provideRouter([])` and `provideNoopAnimations()` (not needed in Angular 22) | The stock spec **fails** with `NullInjectorError` |
| Leaving the CLI's root component spec as generated | Update the title text and `h1` "Hello, ..." checks; call `fixture.detectChanges()` before inspecting the DOM | The stock spec goes stale |
| `styleUrl: './app.component.scss'` pointing at a `.css` file | Point it at the file that exists | It builds under esbuild but **breaks `ng test`** ("Can't resolve ... ?ngResource") |
| Trusting a green `ng build` for a component with no route | Add temporary routes in a scratch copy and build there | `ng build` (esbuild) only type-checks files reachable from `main.ts`; template errors (NG9, TS2741) only appear for compiled components |
| A spec that touches a saved sort without cleaning up | Clear `localStorage` in its `beforeEach` | A sort saved by an earlier spec leaks in |

- Headless: `export CHROME_BIN="/c/Program Files/Google/Chrome/Application/chrome.exe"; npx ng test --watch=false --browsers=ChromeHeadless`.
- Specs for a paged screen (`provideHttpClientTesting()`, `HttpTestingController`): the first request has page 0/1 and the page size; a header click sends the next sort and returns to page 1; a search returns to page 1 and trims the text; a page past the end asks again for the last page. Match with `http.expectOne((r) => r.url.endsWith('/search') && r.params.get('pageNumber') === '4')`.

## Providers (build passes, app breaks)

| Wrong | Right | Why |
|---|---|---|
| Material's `mat-sidenav` with no animations provider, or a cleanup that removes a duplicated provider and leaves none | `provideAnimationsAsync()` (or `provideAnimations()`) before Angular 22; leave **one** | The page renders blank with `NG05105: Unexpected synthetic listener @transform`. Nothing in `ng build` or the stock specs catches this: run the app (see `verify-in-running-app`) |
| An entry in `app.config.ts` for a service that is `providedIn: 'root'` | None needed | Root-provided services need no entry |

## Dev proxy

The dev server proxies `/api` to the API with the `/api` prefix stripped (`proxy.conf.js` reads an env var such as `services__<api>__http__0`); the API's own routes have no `/api` prefix, and `Location` headers built as `"/api" + route` are what the browser sees. Set the env var in the same command that starts `ng serve`.

## Angular 22: OnPush is the default

| Wrong | Right | Why |
|---|---|---|
| A screen that fills `this.rows = data` inside `subscribe`, with no `changeDetection` setting | Set `ChangeDetectionStrategy.Default` (a deprecated alias of `Eager`) on the screen **and** on the root component | Every component without a setting is `OnPush`: the request succeeds, there is no console error, and nothing renders. An OnPush **ancestor** (the root `App`) stops change detection reaching the screen |

`ng new` in 22 defaults to zoneless (pass `--zoneless=false` for zone-based apps) and to `--test-runner=vitest` (jasmine-style `describe/it/expect` specs run unchanged; `npm test -- --watch=false`).

## Layout

| Wrong | Right | Why |
|---|---|---|
| An edit form under a long grid | A native modal `<dialog #editDialog *ngIf="selectedRow" (cancel)="$event.preventDefault(); cancel()">` opened by a setter: `@ViewChild('editDialog') set editDialog(el: ElementRef<HTMLDialogElement> \| undefined) { if (el && !el.nativeElement.open) el.nativeElement.showModal(); }`. Style `dialog.form-container` (width, `max-height: 90vh`, `overflow: auto`, `margin: auto`) and `::backdrop` | The form is off-screen. This needs no library, is centered, and Escape fires `cancel` |
| `<router-outlet />` placed directly in a CSS grid with the menu in column 1 | `<div class="screen"><router-outlet /></div>` | The routed component lands NEXT TO `<router-outlet>`, not inside it, so the outlet is itself a grid item |
| A search bar that is also a `.form-group` | Add a more specific `.form-group.form-group-search { flex-direction: row; flex-wrap: wrap; align-items: center; }` after the generic rules, with a fixed input width | It stacks vertically when global css makes every `.form-group` a column |

## Opening a row of another screen and coming back

The contract (query parameters, route names, child columns) is in `paging-and-grids` (codegen-t4-templates `reference/`). In Angular: `router.navigateByUrl('/sales-invoice?edit=31&back=' + encodeURIComponent('/customer-monthly-summary?edit=1'))`; each screen reads `route.snapshot.queryParamMap` in `ngOnInit`, and `back` calls `router.navigateByUrl(back)` after a successful save or Cancel.

## Server-paged grids

| Wrong | Right | Why |
|---|---|---|
| Taking only `event.pageIndex` from the `(page)` event, or a default page size that is not in `[pageSizeOptions]` | Bind `mat-paginator` `[length]` to the server's total and `[pageIndex]` to your own page number; take **both** `event.pageIndex` and `event.pageSize`; give `[pageSizeOptions]` and make the default size one of them | Ignoring `pageSize` makes the choice do nothing. The server pages, sorts and searches; the screen holds one page |
| A local `filter` after a delete | Read the page again, and fall back when it is past the end: `if (items.length === 0 && count > 0 && pageIndex > 0) { pageIndex = Math.ceil(count / pageSize) - 1; load(); return; }` | A local filter leaves the total, the rows that move up and the last page wrong |
| A sort header that is `<a href="#">` with `preventDefault()` | A `<button type="button">` in the `<th>` with `[attr.aria-sort]`, styled as header text (`background: none; border: 0; font: inherit; color: inherit`) | Keyboard and screen reader access |
| Restoring any saved sort | `loadSort(key, allowedColumns, fallback)`: ignore anything not in the list and any direction other than asc/desc; wrap `localStorage` in try/catch | The server answers an unknown sort column with 400 |
| A grid that only needs names calling the entity list | Call the lookup endpoint (`{ id, name }`); type the rows with a small interface (the full entity satisfies it structurally); add a spec that `http.expectNone('api/employees')` and expects the `/lookup` call | The entity list carries its joins |
| Clear sort only on right-click | `<table (contextmenu)="openSortMenu($event)">` with `event.preventDefault()`, a `sortMenu = { x, y } \| null` rendered as a fixed-position `<ul role="menu">`, closed by `@HostListener('document:click')` and `('document:keydown.escape')`; **also** a visible "Clear sort" button beside the search box, disabled while the sort equals the default. Clearing removes the saved sort, returns to page 1 and reloads | A right-click-only feature is out of reach of the keyboard |

- "Nothing found." only after the first response (a `loaded` flag); the step back after a delete is in `paging-and-grids` (codegen-t4-templates `reference/`).
- A column you can search and sort by should be visible.
- Check it in the browser as a person would: next page, a page size, the last page, delete its only row (the grid lands on the previous page with the right total), search for nothing, reload and see the sort kept. Material's page-size select: `mat-paginator .mat-mdc-select`, options `mat-option`.

## One base class for list-and-form screens

- **`@Directive() abstract class CrudScreen<T>`** (a base using `@HostListener` or `inject()` needs `@Directive()`): `rows`, `selectedRow` (always a copy), `searchText`; `add / edit / submit / delete / cancel / search`. A subclass declares `service = inject(XService)`, `noun`, `idKey`, `newRow()` and overrides small hooks: `prepare(row, isNew)`, `prepareEdit(copy)`, `start()` (extra drop-downs), `find(text)`. `PagedCrudScreen<T> extends CrudScreen<T>` adds the server page, kept sort, arrows and the Clear-sort menu.

| Wrong | Right | Why |
|---|---|---|
| Member names that differ per template | The same names everywhere (`rows`, `selectedRow`, `add()`, `submit(row)`); a component a child binds with `[(selectedTimeSheet)]` keeps that name through a getter/setter over `selectedRow` | One base class serves every screen |
| The base using an abstract property in a field initializer | Use it only in methods (`ngOnInit`, `load`); read the sort in `start()` | Abstract properties are assigned after the base constructor |
| An abstract property left wide in the paged subclass, or a defaulted string field with no type | `protected abstract override readonly service: PagedCrudService<T>`; give the string field an explicit `: string` type | A subclass override otherwise fails with "is not assignable to type" |

- Keep the form open with the typed values when the API refuses a save; one wording per status (400 name or what the API said, 422 already exists, 404 gone so reload, 400 on delete = in use); read the list again after every save and delete.
- Test the base through the smallest screen with `HttpTestingController`; a screen that renders `mat-paginator` needs `provideNoopAnimations()` in a spec before Angular 22.
- Prove each screen in the browser (add, refused add, edit pre-fill, search, delete). A grid over 50,000 rows takes more than 2 s to reload: poll for the new count instead of a fixed sleep, or the script reads the stale grid and deletes the wrong row. A bootstrap admin has no employee id, so adding a request returns 500 until the scratch copy links the user to an employee.
