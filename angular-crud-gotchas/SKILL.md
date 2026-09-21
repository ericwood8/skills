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
