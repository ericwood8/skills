---
name: verify-in-running-app
description: Use when a change touches an ASP.NET API plus a front end (React, Angular, Blazor, WinUI3), a generated screen, a provider or startup code and needs a real click-through against a disposable copy of the database, because a passing build and unit tests miss broken apps. Trigger words - scratch database, click-through, UI Automation, built-in browser, generate --dry-run --diff, sign-in check, dialog crashes on open, "it builds but".
---

# Verify in the running app, on a scratch database copy

A passing build and unit tests do not show that the app renders. Any edit to `app.config.ts`, providers, routes or startup code needs a real run. Create, edit and delete tests write rows: never test against the real database.

The usual order: scratch database, run the stack, drive the screens, read the database back, tear everything down. Find your task in the table, read that one file in `reference/`, and read a second only when the first points to it. Each reference file starts with a "Read this when" line.

## Pick the file

| You are about to... | Read |
|---|---|
| Make, use or drop a disposable database (SQL Server, PostgreSQL, MySQL, Docker) | [scratch-database](reference/scratch-database.md) |
| Start or stop the API and front end, or build a sample that never compiled a template | [run-the-stack](reference/run-the-stack.md) |
| Play the generated API tests (ASP.NET, Python, Rust) over a live SQL Server, PostgreSQL or MySQL with no Docker | [live-database-tests](reference/live-database-tests.md) |
| Click through a React, Angular or Blazor page in the browser pane | [drive-web-page](reference/drive-web-page.md) |
| Decide what a screen test must cover, or check a changed form field, caption or mark | [screen-checklist](reference/screen-checklist.md) |
| Click through a WinUI3 app against a scratch database (the UI Automation techniques are in the `winui3` skill) | [desktop-winui3](reference/desktop-winui3.md) |
| Show that a check passing is not a false pass, or compare old and new code with no behavior change | [proving-the-check](reference/proving-the-check.md) |
| Prove a template change moved no default output (dry-run diff, byte compare with HEAD) | [regression-diff](reference/regression-diff.md) |
| Look for an existing script before writing one | [harness-scripts](reference/harness-scripts.md) |
| Prove sign-in, roles or owner scope over HTTP (ASP.NET, Python, Rust) | [sign-in-and-roles](reference/sign-in-and-roles.md) |
| Prove an update keeps an unchanged stored bad value but refuses a new one | [bad-value-update-checks](reference/bad-value-update-checks.md) |

## Rules that apply to every run

1. **Never test against the real database.** Tell the user you are making a copy (creating a database on their server is an action they should know about), and drop it afterwards ([scratch-database](reference/scratch-database.md)).
2. **Read the database after every write.** The grid can lie (updated client-side) and an un-subscribed HTTP call changes nothing ([drive-web-page](reference/drive-web-page.md)).
3. **Prove a check can fail.** Switch the fix off, see the failure you expect, restore ([proving-the-check](reference/proving-the-check.md)).
4. **Open the screen the change lives on.** A caption, mark or drop-down is on the detail screen (press Edit); the list page, the menu and the build prove nothing about it ([screen-checklist](reference/screen-checklist.md)).
5. **Run the repository's harness; do not rewrite the scaffolding** ([harness-scripts](reference/harness-scripts.md)). From Git Bash an argument starting with `/` is rewritten into a path: set `MSYS_NO_PATHCONV=1` when calling PowerShell.
6. **Stop only what this session started**, by listening port, and say which URLs are up.
7. **Report what you could not run** (a stack with no sample, a browser step skipped) instead of leaving it out.

## When something is missing: pick the next rung and say so

1. A standing SQL Server test database exists (`InvoiceSystem_test`, `TimeEntry_test`): use it ([scratch-database](reference/scratch-database.md)).
2. Otherwise make a scratch copy of the real database (tell the user first).
3. If you may not create a database (a login without CREATE DATABASE), use a SQLite scratch project for the Python, Rust, ASP.NET and Blazor stacks ([sign-in-and-roles](reference/sign-in-and-roles.md)), or the rolled-back procedure test for MySQL.
4. If the stack cannot be started (no Docker, a port in use, no browser pane), run the build and tests only and report the click-through as **not done**. Never report a build as a verification of the screen.

Default when the user did not say: rung 2, a copy named `<db>_scratch`, dropped at the end.

## Report format

End with three short lists, ranked most serious first inside each:

- **Proved** (one line each): what was done, in which stack, and the database evidence (the row read back). 
- **Found** (one line each): `location | what happens | cost | fix`, ranked: wrong or lost data, then a crash on open, then a wrong status code, then a cosmetic fault. A finding with no way to reproduce it is not listed.
- **Not proved**: stacks, screens or paths you did not run, and why. Also say what was left running and what was dropped.

## Related skills

`winui3` (WinUI3 patterns and the crash log), `angular-crud-gotchas`, `react-crud-gotchas`, `odata-generated-grids` (grid click-through), `codegen-t4-templates` (what to regenerate), `mssql`, `mysql`, `postgres`, `docker`, `git-bash-editing`, `angular-major-upgrade`, `dotnet-major-upgrade`.

## Signing in to a test database, and a spare-port stack for an Angular UI

- **A `_test` database may have no sign-in users.** The password hash does not depend on the user, so make one with `python C:/Github/tools/cs_snippet_check.py -e 'using Microsoft.AspNetCore.Identity; Console.WriteLine(new PasswordHasher<object>().HashPassword(new object(), "a-test-password1"));' --package Microsoft.Extensions.Identity.Core@<framework major>.0.0 --run`, then `INSERT` a row (role Admin) into the user table of the `_test` database only, with the login's password from `SQLCMDPASSWORD`. Delete the row when finished and say so.
- **Start the API on a spare port with the connection in the environment** (`ConnectionStrings__<Name>=...`, `ASPNETCORE_ENVIRONMENT=Development`, `--urls http://localhost:5090`), with `python C:/Github/tools/serve_project.py start <API folder> --port 5090 --path /health` (it inherits variables exported in the same command, and waits until the port answers). A bare `dotnet run ... &` inside a `run_in_background` call ends the launcher at once, so its completion notice means nothing; if you must launch that way, wait with a Monitor `until curl -s -o /dev/null http://localhost:5090/health; do sleep 3; done`. Then call the login route with `curl` to prove the account works.
- **Angular dev server through its proxy**: when `proxy.conf.js` reads `services__<api>__http__0`, start `ng serve --port 4299` with that variable set to the API URL and `NODE_ENV=development`. Wait for the port the same way; the first `navigate` while it still builds fails ("denied or failed"), so retry. `curl -X POST http://localhost:4299/api/auth/login` proves the proxy.
- **A collapsed browser pane reports a 0x0 viewport** and `read_page` is empty: `resize_window` 1280x900 first, and reset it with the `desktop` preset at the end.
- **Fill and click the way the framework listens.** A menu item that looks like a list item may carry `routerLink` on an inner element (click `[routerLink]`, not the wrapper). A select bound with `[ngValue]` booleans has no option whose value is `'false'`: set `selectedIndex` and dispatch `change`; setting `.value` sends garbage and the API answers 400 "request body is missing or is not valid JSON", which looks like an app bug and is not.
- **A list may hide inactive rows.** Editing a row to inactive makes it vanish from the grid (and from search), so test Delete on a fresh active row and query the database for the inactive one.
- **Prove each write in the database** (`sqlcmd` after add, edit, delete), then check the network list (`read_network_requests`) for non-2xx calls: the console buffer keeps old errors, so a 400 there may be a test step from minutes ago.
- **Stop what you started**: find each listener with `netstat -ano | grep ":4299 .*LISTENING"` and `taskkill //PID <pid> //T //F` (the dev server is a Node process, which `stop_dev_server.py` leaves alone), the API with `stop_dev_server.py --port 5090`. Confirm no listener is left and the scratch rows are gone (row counts as at the start).
