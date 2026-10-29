# Updates that leave a stored bad value alone

Read this when: you prove that a validation rule lets an unchanged stored bad value through but blocks a new one, in any stack.

## Proving that an update leaves a stored bad value alone

- Put a bad value in a scratch copy (`not-an-email`, a negative amount, a future date). Then four calls per layer: the stored value sent again with another column changed (accepted), the value changed to another bad one (refused), a new bad value on a different row (refused), a good value (accepted). Read the row back after each.
- Layers differ: SQL routine (call it), ASP.NET (`curl -X PUT` with the GET body edited), Python (`uvicorn` with `DB_NAME`, `ODBC_DRIVER`), Rust (`PGDATABASE`, `PORT`; `docs\cargo.cmd run`). A `400` text in FluentValidation style (`'Name' is not ...`) comes from the endpoint filter, not the attributes: a second gate to relax.
- A browser form can still stop it: `type="email"` blocks the submit natively with no request and no message in the DOM; check `window.__puts` (a fetch wrapper that records non-GET calls) before blaming the server.
- Blazor: set an input with the native setter of its prototype, dispatch `input` and `change`, click the Save button, wait about 1.5 s, read `.validation-message`. A child row's Edit opens the child page with `back=` and returns after saving.

## Traps met proving form checks in each stack

- **WinUI3:** the grid's search row has boxes named like the dialog's fields (`Billing Email Address`). Find the dialog window first and search only inside it, after selecting the tab that holds the field (`SelectionItemPattern` on the `TabItem`); a name search over the whole tree types into the filter box and the check "passes" for the wrong reason. The sample's quick regenerate build is not x64: `dotnet build -p:Platform=x64` before the click test, and point the `bin` copy of `appsettings.json` at the scratch database and put it back in `finally`.
- **Angular:** `ng serve` needs `--proxy-config proxy.conf.json` when `angular.json` has no `proxyConfig`; without it `/api/...` answers the index page with 200 and the console shows a parse error with `status: 200, ok: false`. `ng test --watch=false` (no `--browsers`) is the runner that works in the sample.
- **React:** the dev server is `npm run dev` (5173); routes are singular (`/customer?edit=5`). A tabbed form hides fields on other tabs, so read which tab is showing before and after a blocked submit.
- **Proving a relaxed check also needs the strict side:** after the unchanged-value save, change that same field to another bad value (blocked, message visible, nothing sent) and open Add with a required field empty (still blocked).
