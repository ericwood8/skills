---
name: verify-in-running-app
description: How to prove a change works by actually running the API and UI against a disposable COPY of a SQL Server database, driving the UI in the built-in browser and checking the database afterwards — because a passing build and unit tests missed a broken app. Use when a change touches an ASP.NET API plus a front end and needs a real click-through, especially when create/update/delete would otherwise hit a real database.
---

# Verify in the running app, on a scratch database copy

## Why

A build and 12 passing unit tests said "done" while the app rendered a blank page: an edit that "removed the duplicate `provideAnimationsAsync()`" removed **both** copies, and Angular Material's sidenav then threw `NG05105: Unexpected synthetic listener @transform`. Neither `ng build` nor the stock specs exercise app-level providers. Only running the app found it. Any edit to `app.config.ts`, providers, routes, or startup code needs a real run.

## Never write to the real database

Testing add/edit/delete means writing. Make a copy first, point the app at the copy, and drop it afterwards.

1. Find paths: `SELECT SERVERPROPERTY('InstanceDefaultBackupPath'), SERVERPROPERTY('InstanceDefaultDataPath'), SERVERPROPERTY('InstanceDefaultLogPath')` and the source's logical file names (`SELECT name FROM <db>.sys.database_files`).
2. `BACKUP DATABASE [Real] TO DISK = N'<backup path>\x.bak' WITH COPY_ONLY, INIT;` then `RESTORE DATABASE [Real_UITest] FROM DISK = N'...' WITH MOVE N'<logical data>' TO N'<data path>\Real_UITest.mdf', MOVE N'<logical log>' TO N'<log path>\Real_UITest_log.ldf';`
3. Sanity check row counts in both databases.
4. Afterwards: stop the servers, `ALTER DATABASE [Real_UITest] SET SINGLE_USER WITH ROLLBACK IMMEDIATE; DROP DATABASE [Real_UITest];`, confirm `SELECT COUNT(*) FROM sys.databases WHERE name LIKE 'Real%'` = 1, and re-check a few rows of the **real** database (counts, one value you changed in the test) to show it was untouched.
5. The `.bak` sits in a folder the assistant usually cannot delete — say where it is and ask the user to remove it. Do not enable `xp_cmdshell` or use bulk-delete procs.

Tell the user you are making a copy (creating a database on their server is an action they should know about).

## Run the stack

- **API:** set the connection string through environment variables in the *same* command (`ConnectionStrings__<Name>`, and any `Aspire__...__ConnectionString` the app also reads), then `dotnet run --project <Api> --no-launch-profile --urls http://localhost:5084` in the background. Check with `curl` that a GET returns rows from the copy.
- **UI:** `ng serve --port 4200` in the background; if a proxy config reads an env var for the API address (Aspire style `services__<name>__http__0`), export it first. Check `curl http://localhost:4200/api/<route>` returns the same JSON — that proves the proxy.
- Stop both when done: `taskkill //F //PID <pid>` (find pids with `netstat -ano | grep LISTENING`).

## Drive it in the built-in browser

- `navigate`, then run scripts with the JavaScript tool. First override dialogs: `window.__alerts=[]; window.alert = m => window.__alerts.push(String(m)); window.confirm = () => true;` (they reset on every navigation), and read `__alerts` after each action — that is how error paths (400 "bad name", 400 "in use") are seen.
- Angular `ngModel` needs events: set `el.value = v; el.dispatchEvent(new Event('input',{bubbles:true})); el.dispatchEvent(new Event('change',{bubbles:true}))`; click buttons with `.click()`; `await` a sleep (about 1 s) after each server call.
- After every write, **query the database** (`sqlcmd ... -Q "SELECT ..."`) to confirm the row really changed — the grid can lie (it was updated client-side) and an un-subscribed HTTP call changes nothing.
- Screenshots can time out when the window is behind others; use `document.querySelector(...).innerText` or `get_page_text` instead. The console log buffer keeps old errors from earlier page loads — reload before concluding an error is current.
- Choose test values that satisfy the server's own validation (this API rejected a name containing parentheses as "special characters"); a rejected test proves the error path but not the save path, so follow it with a valid one.

## What to test per screen

Grid loads with real names/dates; edit form pre-fills correctly (dates, booleans, drop-downs); update, add, delete, search each reach the database; Cancel closes the form; the "in use" delete path alerts; an invalid value alerts. Finish with `ng build` and `ng test --watch=false --browsers=ChromeHeadless` (see the `angular-crud-gotchas` skill).


## Notes from checking web screens in the built-in browser

- A screenshot may time out ("the page did not finish rendering") when the app window is behind another; do not wait on it. `javascript_exec` with `getBoundingClientRect()`, `getComputedStyle` and `document.querySelectorAll` answers most layout questions (is the dialog inside the viewport after `scrollTo(0, 800)`; are the search boxes on one row = same `top`; did the date box empty after "Clear").
- A script that sets `location.href` is cut off; navigate with the navigate tool, then run a second script.
- To prove a save, read the row back with `sqlcmd` (a cleared date must be `NULL`, not an empty string), then restore the value.
- You start servers, you stop them: kill by listening port (`Get-NetTCPConnection -LocalPort ...`) before a regenerate/build, and say in the answer which URLs are up.


## Notes from checking a desktop (WinUI3) app and a second database

- **Drive a WinUI3 app with UI Automation from PowerShell** instead of screenshots: `Start-Process` the built exe (set any password environment variable in the same script first, so it is not on a command line), wait ~6 s, find the window by process id (`AutomationElement.RootElement.FindFirst(Children, ProcessIdProperty = pid)`), find a menu item or button by `NameProperty`, `InvokePattern.Invoke()` (or `SelectionItemPattern.Select()`), wait ~4 s, then list `Descendants` names: a loaded grid shows its row cells (`TES00001 | PAC62163 | 09/19/2026 | Edit | Delete`), an empty one only headers and "Page 1 of 1". A ComboBox: `ExpandCollapsePattern.Expand()`, then select the item by name. Kill the process at the end. Use the exe in `bin\x64\Debug\...` (a stale `bin\Debug` one misleads). A first launch can show an empty grid when a previous instance is still shutting down: rerun before concluding anything.
- **The built-in browser pane can be collapsed to almost no width**: every element then measures `width 0` and layout checks "fail" (a flex row looked 20 px wide). Set the viewport first (`resize_window` 1280x900), measure, then reset it with the `desktop` preset.
- **A measure inside a closed or hidden `<dialog>` or inactive tab reads zeros too**: find the open one (`dialog[open]`) and check ancestors for `hidden`/`display:none` before believing a size.
- **Same front end, another database**: when every database sample serves the same routes and JSON, prove a new database by stopping the other API on that port and starting the new API on 5080 (React) / 5081 (Angular) with the credentials in an environment variable; the existing front end then reads the new database unchanged. Say which API owns the port when you finish.
- **Slow regenerate scripts** (one CLI run per file, minutes) go in the background with a `done` marker line and a Monitor on it; a foreground call is cut at two minutes. Do not start another build of the same project while one runs (the CLI exe and `obj` are shared).


## More checks that paid off (database-level templates, drop-downs, junction routines)

- **Stopping a process by its command line**: `Get-CimInstance Win32_Process | Where CommandLine -like '*--urls http://localhost:509*'` also matches the **shell that is running the command** (its own command line contains the pattern), so the filter killed the Bash tool's own processes and the rest of the script never ran. Find the listener by port instead (`netstat -ano | grep ":5080 .*LISTENING"`), print that pid's `Path` / `CommandLine` first, and only then `taskkill //PID`. Stop only processes this session started.
- **A scratch copy of a sample API** is the cheap way to try a generated file against a live database: copy the project without `bin`/`obj` into the scratchpad, drop the generated files in, build, run it on a spare port with `--urls http://localhost:5091` (a `Urls` entry in `appsettings.json` otherwise wins), call it with `curl`, then delete the copy. The sample itself stays untouched. For a generated front-end page do the same inside the sample but with a temporary route (`App.tsx` / `app.routes.ts`), keep a backup of the route file, run `npx tsc -b` (or `npx ng build`) and the page's own vitest file, and revert.
- **Filling a React controlled input from the console**: assigning `el.value = ...` is ignored. Use the native setter of the element's own prototype and dispatch the event: `Object.getOwnPropertyDescriptor(HTMLInputElement.prototype, 'value').set.call(el, v)` (`HTMLTextAreaElement` for a `<textarea>`, `HTMLSelectElement` for a `<select>`), then `el.dispatchEvent(new Event('input' | 'change', { bubbles: true }))`. Using the wrong prototype throws "Illegal invocation" (a text field generated as a `<textarea>` looks like an input).
- **A hidden or minimized browser pane cannot take clicks** ("Screenshot timed out", "not on screen"): drive the page with `javascript_tool` (click buttons, read `select.options`, `fetch` the API) and read the DOM instead of screenshots.
- **UI Automation for a WinUI3 ComboBox**: `SelectionPattern.Current.GetSelection()` gives the selected item's name (the `ValuePattern` value is empty); `ExpandCollapsePattern.Expand()` then `FindAll(Descendants, ControlType == ListItem)` lists the options. A sample's exe may not be named after the project (`InvoiceSystem.exe`, not `InvoiceSystem.App.exe`): search `bin` for `*.exe` before scripting a path.
- **A sample that has never been compiled with a template's output hides template bugs**: building the generated WinUI3 junction editor in a real app found a `--` inside a XAML comment that no unit test had caught. When a template has never been built in a sample, build it once, and add a test that parses the generated XAML as XML (`XDocument.Parse`).
