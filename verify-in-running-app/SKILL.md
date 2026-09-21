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
