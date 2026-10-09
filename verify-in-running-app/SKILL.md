---
name: verify-in-running-app
description: How to prove a change works by running the API and UI (web or WinUI3) against a disposable COPY of the database, driving the screens, and checking the database afterwards, because a passing build and unit tests miss broken apps. Covers scratch copies for SQL Server, PostgreSQL, MySQL and Docker, starting the stack, driving the built-in browser and UI Automation, the per-screen checklist, proving a check can fail, and regression checks with generate --dry-run --diff. Use when a change touches an ASP.NET API plus a front end, a generated screen, a provider or startup code and needs a real click-through.
---

# Verify in the running app, on a scratch database copy

A build plus passing unit tests has called an app "done" while it rendered a blank page (an edit meant to remove one duplicate provider removed both, and Material threw `NG05105`). Any edit to `app.config.ts`, providers, routes or startup code needs a real run. Create, edit and delete tests write rows: never test against the real database.

## Scratch database copy

Tell the user you are making a copy (creating a database on their server is an action they should know about). The harness does it: `Docs/Verification/ScratchDatabase.ps1 -Provider SqlServer|PostgreSql|MySql -Source <db> [-Drop]` (creates `<db>_scratch`).

- **SQL Server:** backup `WITH COPY_ONLY, INIT` and restore with `MOVE` to a new name. The SQL Server **service** writes the backup, so use `SERVERPROPERTY('InstanceDefaultBackupPath')`, not a user temp folder. Get logical file names from `RESTORE FILELISTONLY`. A failing native command does not stop a PowerShell script: check `$LASTEXITCODE`.
- **PostgreSQL:** stop every connection to the source (a running API holds a pooled one), then `CREATE DATABASE "X_scratch" TEMPLATE "X";` (needs CREATEDB). No dump or `.bak` is left.
- **MySQL:** copy the tables (`CREATE TABLE ... LIKE` + `INSERT ... SELECT`); a login limited to one database cannot create one.
- **A throwaway Linux database:** `docker run -d --rm --name <name> -e MYSQL_ROOT_PASSWORD=<throwaway> -p 127.0.0.1:3307:3306 mysql:8.4` on a spare port (never the user's containers); wait with `mysqladmin ping`; load the schema with `docker exec -i`. Docker Desktop's engine can stop by itself, so do every check in one sitting.
- Sanity-check row counts in both databases. Passwords go in `PGPASSWORD` / `MYSQL_PWD` / environment connection strings, never a file or parameter.
- **Afterwards:** stop the servers, drop the copy (`ALTER DATABASE ... SET SINGLE_USER WITH ROLLBACK IMMEDIATE; DROP DATABASE`), confirm only the real database remains, and re-check counts and one changed value in the **real** database. Remove your `.bak` from the backup folder if you can (`rm` worked there); otherwise tell the user where it is. Never enable `xp_cmdshell`.

## Run the stack

- **API:** set `ConnectionStrings__<Name>` (and any `Aspire__...__ConnectionString`) in the same command, then `dotnet run --project <Api> --no-launch-profile --urls http://localhost:5084` in the background (a `Urls` entry in `appsettings.json` otherwise wins). A first admin on an empty user table needs `Bootstrap__AdminUserName` / `Bootstrap__AdminPassword` / `Jwt__Key`. Confirm with `curl` that a GET returns rows from the copy.
- **UI:** `ng serve --port 4200` in the background; export the API address variable a proxy config reads (`services__<name>__http__0=http://localhost:5084`) first. `curl http://localhost:4200/api/<route>` returning the same JSON proves the proxy.
- **Rebuild only after stopping the API** (`MSB3027` file lock). `dotnet run` starts a child process: stop with `taskkill /PID <id> /T /F`.
- **Stop processes by listening port** (`netstat -ano | grep ":5084 .*LISTENING"` or `Get-NetTCPConnection`), printing the pid's path first. Filtering `Get-CimInstance Win32_Process` by command line also matches the shell running the command and kills it. Stop only what this session started and say which URLs are up.
- **A sample never compiled with a template hides template bugs:** build it once. Copy the sample without `bin`/`obj` to the scratchpad, drop the generated files in, build, run on a spare port (`--urls http://localhost:5091`), curl it, delete the copy. For a front-end page add a temporary route (`App.tsx` / `app.routes.ts`) after backing up the route file, run `npx tsc -b` or `npx ng build` and the page's vitest file, then revert.
- Another database, same front end: stop the other API on that port and start the new one there (React 5080, Angular 5081); the front end reads it unchanged. Say which API owns the port.

## Drive a web page in the built-in browser

- Override dialogs first (they reset on every navigation): `window.__alerts=[]; window.alert = m => window.__alerts.push(String(m)); window.confirm = () => true;`, and read `__alerts` after each action; that is how error paths (400 bad value, 400 in use) are seen.
- **Angular `ngModel`:** set `el.value = v; el.dispatchEvent(new Event('input',{bubbles:true})); el.dispatchEvent(new Event('change',{bubbles:true}))`; click with `.click()`; `await` about 1 s after each server call.
- **React controlled input:** use the native setter of the element's own prototype (`HTMLInputElement`, `HTMLTextAreaElement` or `HTMLSelectElement`; the wrong one throws "Illegal invocation"): `Object.getOwnPropertyDescriptor(HTMLInputElement.prototype,'value').set.call(el, v)`, then dispatch `input` / `change`. A text field generated as a `<textarea>` looks like an input.
- **After every write, query the database** (`sqlcmd`, `psql`) to confirm the row changed; the grid can lie (updated client-side) and an un-subscribed HTTP call changes nothing. A cleared date must be `NULL`, not an empty string.
- Screenshots time out when the window is behind others; prefer `innerText`, `get_page_text`, `getBoundingClientRect()`, `getComputedStyle`, `querySelectorAll`, `fetch`. A hidden or minimized pane cannot take clicks: drive with `javascript_tool`.
- **Set the viewport first** (`resize_window` 1280x900) and reset it with the `desktop` preset; a collapsed pane makes every element measure width 0. A closed `<dialog>` or hidden tab also measures zeros: find `dialog[open]` and check ancestors for `hidden`.
- A script that sets `location.href` is cut off: use the navigate tool, then a second script. The console buffer keeps errors from earlier loads: reload before concluding an error is current. Editing an HTML file can move the pane to the local file: navigate back to the app URL, and do the browser step after the edits. The token survives `navigate` within the app; log in through the form once.
- Use test values the server's own validation accepts (a name with parentheses was rejected as "special characters"); a rejected value proves the error path only, so follow it with a valid one.

## What to test per screen

Grid loads with real names and dates; the edit form pre-fills (dates, booleans, drop-downs); update, add, delete and search each reach the database; Cancel closes the form; the "in use" delete and an invalid value alert; **walk the Add path of every form, not just Edit** (a required foreign-key `<select>` starting at 0 shows the first parent in the browser but sends 0, the database refuses it and the page says only "Problem while saving"). Make the cases the screen must handle: a page with a single row (insert an 11th row so page 2 holds one, then delete it), an empty search, a reload after a sort, a stale value in `localStorage`. Read the rows, paginator text and the headers' `aria-sort` back from the DOM. Finish with `ng build` and `ng test --watch=false --browsers=ChromeHeadless` (see angular-crud-gotchas).

## Desktop (WinUI3) apps

- Drive with UI Automation from PowerShell, not screenshots: `Start-Process` the built exe (set password environment variables in the same script), wait about 6 s, find the window by process id, find controls by `NameProperty`, `InvokePattern.Invoke()` / `SelectionItemPattern.Select()`, wait about 4 s, list `Descendants` names. A loaded grid shows its row cells; an empty one only headers and "Page 1 of 1". Kill the process at the end. A first launch can show an empty grid if a previous instance is still shutting down: rerun before concluding.
- Use the exe in `bin\x64\Debug\...` (search `bin` for `*.exe`; the name may differ from the project's), not a stale `bin\Debug` one.
- ComboBox: `SelectionPattern.Current.GetSelection()` gives the selected item (the `ValuePattern` value is empty); `ExpandCollapsePattern.Expand()` then `FindAll(Descendants, ListItem)` lists options. TabItem: `SelectionItemPattern.Select()`, then list `Edit` controls by `Name`.
- Dialog traps: a TextBox commits on focus loss (`SetValue`, then `SetFocus` on the Search button, then `Invoke`); a grid far down a tall dialog has an empty `BoundingRectangle` until you `SetScrollPercent(NoScroll, 100)` its scroll ancestor (`Invoke` works off-screen, a real click does not); a header `Button`'s `Name` includes the sort arrow (`Customer PO ▼`), so match `-like "Customer PO*"`; a ContentDialog that keeps itself open can be clicked again.
- Read a cell after a formatting change (`CurrencyFormatter` printed `$7777.77` until `IsGrouped = true`). A WinUI3 app reads `appsettings.json` from `bin`: edit that copy to point at the scratch database and put it back in `finally`.
- Add a test that parses generated XAML as XML (`XDocument.Parse`) when a template has never been built in a sample.

## Proving the check itself

- **Prove a check can fail:** after it passes, switch the fix off in the code under test (`if (false && ...)`), rebuild, run the check, look for the failure you expect, restore from a copy taken first and rebuild. Say in the notes that this was done.
- **Refactor with no behavior change:** write checks for what the script does not cover first (roles, for example), then run the same script on the old and new code: `git worktree add <scratch>/oldrepo HEAD`, build it, run with the worktree as the server repo; the old code must pass; then run the new code (same count, all pass); `git worktree remove --force` and `git worktree prune`. Check roles with real sign-ins (admin creates a manager, a report and an unrelated employee; assert 403/200 per route; capture `Location` in the script's call helper).
- **Case sensitivity:** test with two tables that differ only in case (`Movies`, `movies`) and write outputs to **distinct folders** (a Windows folder is case-insensitive, so one overwrote the other and looked like a generator bug).

## Whole-project regression check

After a template change run `codegen generate --project <sample> -o <sample folder> --dry-run --diff` over each sample's database: every file not byte-identical (line endings ignored) is listed with its diff. Expect zero differences for an unrelated change; a wanted change shows exactly where it lands. Then run the sample's `Regenerate.sh` (generate, build, tests). `.codegen-manifest.json` in the sample root tracks files: a dropped table's old files are stale; `--delete-stale` removes those you did not edit. Slow regenerate scripts go in the background with a `done` marker and a Monitor (a foreground call is cut at two minutes); do not start another build of the same project while one runs (the CLI exe and `obj` are shared).

## The harness is in the repository

`Docs/Verification` holds scripts for the above: `ScratchDatabase.ps1`, `UiAutomation.psm1` (`Start-App`, `Find-Name`, `Find-Like`, `Find-Type`, `Wait-Name`, `Invoke-El`, `Select-El`, `Set-Text`, `Assert-That`, `Stop-App`), `Test-AppDialogs.ps1`, `Test-ApiCrud.ps1`, `Test-WinUI3Paging.ps1`, `Test-AngularVersion.ps1` and a recipe per stack. Run them rather than rewriting the scaffolding. From Git Bash an argument starting with `/` is rewritten into a path: set `MSYS_NO_PATHCONV=1` when calling PowerShell. A scratch copy needs the right to create a database.
