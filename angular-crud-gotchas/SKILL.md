---
name: angular-crud-gotchas
description: Concrete Angular (standalone components, Material, ng test/ng build) bugs and fixes found in a CRUD screen app — boolean selects posting the string "true", date boxes and the YYYY week-year format, HTTP calls that never fire, the deprecated two-callback subscribe, missing HttpClient in specs, Material needing animations, esbuild not type-checking unrouted components, headless Chrome test setup, and JSON name casing. Use when writing, generating or reviewing Angular components/services/specs that talk to an ASP.NET API, or when ng test/ng build results look wrong.
---

# Angular CRUD screens: bugs to look for

## Forms and data

- **Boolean `<select>`**: `<option value="true">` binds the **string** `"true"` (and the control shows blank for a real boolean). Use `<option [ngValue]="true">`/`[ngValue]="false"`, or a checkbox. The API's bool binding rejects `"true"` strings.
- **Dates**: JSON has no date type — the API sends and accepts `"2025-12-25T00:00:00"`. Type date properties as `string` (not `Date`). `<input type="date">` needs `yyyy-MM-dd`, so trim in `edit()`: `row.date = row.date.substring(0, 10)`; display with `| date:'MM/dd/yyyy'`. A hand-written `DatePipe.transform(x, 'YYYY-MM-dd')` is a bug: `YYYY` is the *week-based* year (wrong for late December/early January) — use `yyyy`.
- Edit a **copy** (`{ ...row }`) and change the copy; mutating the row modifies what the grid shows.
- **Do not hard-code sample defaults** ("Christmas", "USA") in `add()`; use ''/0/today/the database default.
- Property names must match the server's JSON casing exactly (`sY_RequestStatusTypeId`, `e_TimeSheetId`); a wrongly-cased interface property reads `undefined`.
- `?.` on a value that `*ngIf` already narrowed triggers NG8107 warnings; write `selectedRow.id`, not `selectedRow?.id`, inside the `*ngIf="selectedRow"` block.
- Errors: 400 = the API rejected the value (name with special characters; delete of a row in use -> `BadRequest`), 404 = gone, else show `error.message`. An error handler that only `console.log`s leaves the user with no feedback.

## RxJS

- HTTP observables are **cold**: `service.delete(id)` without `.subscribe(...)` sends nothing. (A Delete button that "did nothing" was exactly this.)
- `subscribe(nextFn, errorFn)` is deprecated; write `subscribe({ next: ..., error: ... })`. A script can rewrite it: find each `.subscribe(` with two top-level arguments, balance parentheses and skip string literals and `//` comments, then re-indent the bodies by two spaces.
- Two independent loads that must combine (time sheets need the employee list to show names) race; chain them (load employees, then in its callback load sheets).

## Tests and builds

- The CLI's stock `should create` spec **fails** for any component whose service uses `HttpClient` (`NullInjectorError: No provider for HttpClient`). Give every such spec `providers: [provideHttpClient(), provideHttpClientTesting()]` (`@angular/common/http` and `.../testing`); components with Material or `RouterLink` also need `provideRouter([])` and `provideNoopAnimations()`.
- The root component spec from the CLI template (`title` equals 'timeentryUI', an `h1` "Hello, ...") goes stale — update it to what the component really does.
- `styleUrl: './app.component.scss'` pointing at a `.css` file builds under esbuild but **breaks `ng test`** (karma/webpack: "Can't resolve ... ?ngResource").
- Run tests headless: `export CHROME_BIN="/c/Program Files/Google/Chrome/Application/chrome.exe"; npx ng test --watch=false --browsers=ChromeHeadless`.
- **`ng build` (esbuild) only type-checks files reachable from `main.ts`.** Generated components with no route are never compiled. To check them, add temporary routes in a scratch copy of the app and build there; template errors (NG9, TS2741 "property missing") only appear for compiled components.

## Providers (build passes, app breaks)

- Angular Material's `mat-sidenav` needs `provideAnimationsAsync()` (or `provideAnimations()`); without it the page renders blank with `NG05105: Unexpected synthetic listener @transform`. A "cleanup" that deletes a duplicated provider must leave **one**. Nothing in `ng build` or the stock specs catches this — run the app (see `verify-in-running-app`).
- Services marked `providedIn: 'root'` need no entry in `app.config.ts` providers.

## Dev proxy

The dev server proxies `/api` to the API with the `/api` prefix stripped (`proxy.conf.js` reads an env var such as `services__timeentryapi__http__0`); the API's own routes have no `/api` prefix, and `Location` headers built as `"/api" + route` are what the browser sees. Set the env var in the same command that starts `ng serve`.

## Angular 22: OnPush is the default, and it breaks "plain property in a subscribe callback" screens

In Angular 22 every component without a `changeDetection` setting is `OnPush`. A screen that fills `this.rows = data` inside `subscribe` then renders **nothing** (the request succeeds, the component
holds the data, the view stays empty, no console error). `ChangeDetectionStrategy.Default` (now deprecated alias of `Eager`) on the screen is not enough: an OnPush **ancestor** (the root `App`)
stops change detection from reaching it, so set `changeDetection: ChangeDetectionStrategy.Default` on the root component too. `ng new` in 22 also defaults to zoneless; pass
`--zoneless=false` for zone-based apps, and `--test-runner=vitest` is the default (the jasmine-style `describe/it/expect` specs run unchanged; run with `npm test -- --watch=false`).
A root spec must call `fixture.detectChanges()` before it inspects the DOM.


## An edit form that opens at the bottom of a long grid: use a native modal `<dialog>`

A form rendered under the grid is off-screen on a long list, and the only sign is a longer scrollbar. Render it as `<dialog #editDialog *ngIf="selectedRow" (cancel)="$event.preventDefault(); cancel()">` and open it as soon as it exists with a setter:
`@ViewChild('editDialog') set editDialog(el: ElementRef<HTMLDialogElement> | undefined) { if (el && !el.nativeElement.open) el.nativeElement.showModal(); }`. No library, centered over the page wherever it is scrolled, Escape fires `cancel`. Style `dialog.form-container` (width, `max-height: 90vh`, `overflow: auto`, `margin: auto`) and `::backdrop`.

## The router puts the routed component NEXT TO `<router-outlet>`, not inside it

In a CSS grid with the menu in column 1, the outlet itself is a grid item and the component lands in the next cell (under the menu). Wrap the outlet: `<div class="screen"><router-outlet /></div>`.

## A search bar that is also a `.form-group`

If global css makes every `.form-group` a column with full-width inputs, the search bar (inputs + Search + Clear) stacks vertically. Add a more specific `.form-group.form-group-search { flex-direction: row; flex-wrap: wrap; align-items: center; }` after the generic rules, with a fixed input width.

## Opening a row of another screen and coming back

Master-detail grids open a child row on its own screen with `router.navigateByUrl('/sales-invoice?edit=31&back=' + encodeURIComponent('/customer-monthly-summary?edit=1'))`. Each screen reads `route.snapshot.queryParamMap` in `ngOnInit` (`edit` -> `getById(...)` -> `edit(row)`; `back` -> `router.navigateByUrl(back)` after a successful save or Cancel), so the person lands back on the parent's open dialog. The route paths are the kebab-case of the table names, a contract with the hand-written `app.routes.ts`.
The child grid's columns come from the JSON keys, which include every navigation property (`customer: null`); restrict them to the child table's own column names.
