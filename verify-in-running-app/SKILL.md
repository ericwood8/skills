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

`winui3` (WinUI3 patterns and the crash log), `angular-crud-gotchas`, `react-crud-gotchas`, `odata-generated-grids` (grid click-through), `codegen-t4-templates` (what to regenerate), `mssql`, `mysql`, `postgres`, `docker`, `git-bash-editing`.
