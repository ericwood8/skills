---
name: angular-crud-gotchas
description: Rules and fixes for Angular CRUD screens over an ASP.NET API (standalone components, Material, ng test/ng build) — boolean selects posting "true", date boxes and the YYYY week-year format, cold HTTP calls, two-callback subscribe, missing providers in specs, Material animations, esbuild not type-checking unrouted components, headless Chrome, JSON casing, OnPush default in Angular 22, modal dialog forms, router outlet layout, server-paged grids, sort headers, lookup lists, and one base class for list-and-form screens. Use when writing, generating or reviewing Angular components/services/specs that talk to an ASP.NET API, or when ng test/ng build results look wrong.
---

# Angular CRUD screens: bugs to look for

## Forms and data

- **Boolean `<select>`**: `<option value="true">` binds the **string** `"true"` (and shows blank for a real boolean). Use `<option [ngValue]="true">`/`[ngValue]="false"`, or a checkbox. The API's bool binding rejects `"true"` strings.
- **Dates**: JSON has no date type; the API sends and accepts `"2025-12-25T00:00:00"`. Type date properties as `string`. `<input type="date">` needs `yyyy-MM-dd`, so trim in `edit()`: `row.date = row.date.substring(0, 10)`; display with `| date:'MM/dd/yyyy'`. In `DatePipe.transform(x, 'YYYY-MM-dd')` `YYYY` is the *week-based* year (wrong in late December/early January): use `yyyy`.
- Edit a **copy** (`{ ...row }`); mutating the row changes what the grid shows.
- Do not hard-code sample defaults in `add()`; use ''/0/today/the database default.
- Property names must match the server's JSON casing exactly (`sY_RequestStatusTypeId`, `e_TimeSheetId`); a wrongly cased interface property reads `undefined`.
- `?.` on a value `*ngIf` already narrowed gives NG8107: write `selectedRow.id` inside `*ngIf="selectedRow"`.
- Errors: 400 = the API rejected the value (special characters; delete of a row in use), 404 = gone, else show `error.message`. A handler that only `console.log`s gives the user no feedback.

## RxJS

- HTTP observables are **cold**: `service.delete(id)` without `.subscribe(...)` sends nothing.
- `subscribe(nextFn, errorFn)` is deprecated: `subscribe({ next: ..., error: ... })`. To rewrite many, find each `.subscribe(` with two top-level arguments, balance parentheses, skip string literals and `//` comments, and re-indent the bodies by two spaces.
- Two independent loads that must combine (sheets need the employee list to show names) race: chain them.

## Tests and builds

- The CLI's stock `should create` spec **fails** for a component whose service uses `HttpClient` (`NullInjectorError`). Give such specs `providers: [provideHttpClient(), provideHttpClientTesting()]`; components with Material or `RouterLink` also need `provideRouter([])` and `provideNoopAnimations()` (not needed in Angular 22).
- The CLI's root component spec (title text, `h1` "Hello, ...") goes stale: update it. A root spec must call `fixture.detectChanges()` before inspecting the DOM.
- `styleUrl: './app.component.scss'` pointing at a `.css` file builds under esbuild but **breaks `ng test`** ("Can't resolve ... ?ngResource").
- Headless: `export CHROME_BIN="/c/Program Files/Google/Chrome/Application/chrome.exe"; npx ng test --watch=false --browsers=ChromeHeadless`.
- **`ng build` (esbuild) only type-checks files reachable from `main.ts`.** Components with no route are never compiled: add temporary routes in a scratch copy and build there; template errors (NG9, TS2741) only appear for compiled components.
- Clear `localStorage` in the `beforeEach` of any spec that touches a saved sort.
- Specs for a paged screen (`provideHttpClientTesting()`, `HttpTestingController`): the first request has page 0/1 and the page size; a header click sends the next sort and returns to page 1; a search returns to page 1 and trims the text; a page past the end asks again for the last page. Match with `http.expectOne((r) => r.url.endsWith('/search') && r.params.get('pageNumber') === '4')`.

## Providers (build passes, app breaks)

- Material's `mat-sidenav` needs `provideAnimationsAsync()` (or `provideAnimations()`) before Angular 22; without it the page renders blank with `NG05105: Unexpected synthetic listener @transform`. A cleanup that removes a duplicated provider must leave **one**. Nothing in `ng build` or the stock specs catches this: run the app (see `verify-in-running-app`).
- Services `providedIn: 'root'` need no entry in `app.config.ts`.

## Dev proxy

The dev server proxies `/api` to the API with the `/api` prefix stripped (`proxy.conf.js` reads an env var such as `services__<api>__http__0`); the API's own routes have no `/api` prefix, and `Location` headers built as `"/api" + route` are what the browser sees. Set the env var in the same command that starts `ng serve`.

## Angular 22: OnPush is the default

Every component without a `changeDetection` setting is `OnPush`. A screen that fills `this.rows = data` inside `subscribe` then renders **nothing** (request succeeds, no console error). `ChangeDetectionStrategy.Default` (a deprecated alias of `Eager`) on the screen is not enough: an OnPush **ancestor** (the root `App`) stops change detection reaching it, so set it on the root component too. `ng new` in 22 defaults to zoneless (pass `--zoneless=false` for zone-based apps) and to `--test-runner=vitest` (jasmine-style `describe/it/expect` specs run unchanged; `npm test -- --watch=false`).

## Layout

- **An edit form under a long grid is off-screen:** use a native modal `<dialog #editDialog *ngIf="selectedRow" (cancel)="$event.preventDefault(); cancel()">` opened by a setter: `@ViewChild('editDialog') set editDialog(el: ElementRef<HTMLDialogElement> | undefined) { if (el && !el.nativeElement.open) el.nativeElement.showModal(); }`. No library, centered, Escape fires `cancel`. Style `dialog.form-container` (width, `max-height: 90vh`, `overflow: auto`, `margin: auto`) and `::backdrop`.
- **The routed component lands NEXT TO `<router-outlet>`, not inside it.** In a CSS grid with the menu in column 1 the outlet is itself a grid item: wrap it, `<div class="screen"><router-outlet /></div>`.
- **A search bar that is also a `.form-group`** stacks vertically when global css makes every `.form-group` a column. Add a more specific `.form-group.form-group-search { flex-direction: row; flex-wrap: wrap; align-items: center; }` after the generic rules, with a fixed input width.

## Opening a row of another screen and coming back

Master-detail grids open a child row on its own screen with `router.navigateByUrl('/sales-invoice?edit=31&back=' + encodeURIComponent('/customer-monthly-summary?edit=1'))`. Each screen reads `route.snapshot.queryParamMap` in `ngOnInit` (`edit` calls `getById(...)` then `edit(row)`; `back` calls `router.navigateByUrl(back)` after a successful save or Cancel), so the person lands back on the parent's open dialog. Route paths are the kebab-case of the table names, a contract with the hand-written `app.routes.ts`. The child grid's columns come from the JSON keys, which include every navigation property (`customer: null`): restrict them to the child table's own column names.

## Server-paged grids

- **The server pages, sorts and searches; the screen holds one page.** Bind `mat-paginator` `[length]` to the server's total and `[pageIndex]` to your own page number; take **both** `event.pageIndex` and `event.pageSize` from the `(page)` event (ignoring `pageSize` makes the choice do nothing). Give `[pageSizeOptions]` and make the default size one of them.
- **A delete reads the page again** (a local `filter` leaves the total, the rows that move up and the last page wrong), and a page past the end falls back: `if (items.length === 0 && count > 0 && pageIndex > 0) { pageIndex = Math.ceil(count / pageSize) - 1; load(); return; }`.
- Say "Nothing found." only after the first response (a `loaded` flag).
- **A sort header is a `<button type="button">` in the `<th>` with `[attr.aria-sort]`**, not `<a href="#">` with `preventDefault()` (keyboard, screen reader). Style it as header text (`background: none; border: 0; font: inherit; color: inherit`).
- **Remember the sort between visits, but only a column the grid can still sort by:** the server answers an unknown sort column with 400. `loadSort(key, allowedColumns, fallback)` ignores anything not in the list and any direction other than asc/desc; wrap `localStorage` in try/catch.
- A column you can search and sort by should be visible.
- **Right-click "Clear sort":** `<table (contextmenu)="openSortMenu($event)">` with `event.preventDefault()`, a `sortMenu = { x, y } | null` rendered as a fixed-position `<ul role="menu">`, closed by `@HostListener('document:click')` and `('document:keydown.escape')`. Also put a visible "Clear sort" button beside the search box (a right-click-only feature is out of reach of the keyboard), disabled while the sort equals the default. Clearing removes the saved sort, returns to page 1 and reloads.
- **A grid that only needs names should call the lookup endpoint** (`{ id, name }`), not the entity list with its joins. Type the rows with a small interface (the full entity satisfies it structurally); add a spec that `http.expectNone('api/employees')` and expects the `/lookup` call.
- Check it in the browser as a person would: next page, a page size, the last page, delete its only row (the grid lands on the previous page with the right total), search for nothing, reload and see the sort kept. Material's page-size select: `mat-paginator .mat-mdc-select`, options `mat-option`.

## One base class for list-and-form screens

- **`@Directive() abstract class CrudScreen<T>`** (a base using `@HostListener` or `inject()` needs `@Directive()`): `rows`, `selectedRow` (always a copy), `searchText`; `add / edit / submit / delete / cancel / search`. A subclass declares `service = inject(XService)`, `noun`, `idKey`, `newRow()` and overrides small hooks: `prepare(row, isNew)`, `prepareEdit(copy)`, `start()` (extra drop-downs), `find(text)`. `PagedCrudScreen<T> extends CrudScreen<T>` adds the server page, kept sort, arrows and the Clear-sort menu.
- **Give every template the same member names** (`rows`, `selectedRow`, `add()`, `submit(row)`); a component a child binds with `[(selectedTimeSheet)]` keeps that name through a getter/setter over `selectedRow`.
- **Abstract properties are assigned after the base constructor**: the base may use them only in methods (`ngOnInit`, `load`), never in field initializers (read the sort in `start()`).
- **Narrow an abstract property in the paged subclass** (`protected abstract override readonly service: PagedCrudService<T>`), and give a defaulted string field an explicit `: string` type or a subclass override fails with "is not assignable to type".
- Keep the form open with the typed values when the API refuses a save; one wording per status (400 name or what the API said, 422 already exists, 404 gone so reload, 400 on delete = in use); read the list again after every save and delete.
- Test the base through the smallest screen with `HttpTestingController`; a screen that renders `mat-paginator` needs `provideNoopAnimations()` in a spec before Angular 22.
- Prove each screen in the browser (add, refused add, edit pre-fill, search, delete). A grid over 50,000 rows takes more than 2 s to reload: poll for the new count instead of a fixed sleep, or the script reads the stale grid and deletes the wrong row. A bootstrap admin has no employee id, so adding a request returns 500 until the scratch copy links the user to an employee.
