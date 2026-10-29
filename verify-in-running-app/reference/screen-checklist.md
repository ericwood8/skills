# What to test per screen

Read this when: you decide what a click-through of a screen must cover, or a change touches a form field, caption, mark or drop-down.

Grid loads with real names and dates; the edit form pre-fills (dates, booleans, drop-downs); update, add, delete and search each reach the database; Cancel closes the form; the "in use" delete and an invalid value alert; **walk the Add path of every form, not just Edit** (a required foreign-key `<select>` starting at 0 shows the first parent in the browser but sends 0, so the database refuses it). Make the cases the screen must handle: a page with a single row (insert an 11th row so page 2 holds one, then delete it), an empty search, a reload after a sort, a stale value in `localStorage`. Read the rows, paginator text and the headers' `aria-sort` back from the DOM. Finish with `ng build` and `ng test --watch=false --browsers=ChromeHeadless` (see angular-crud-gotchas).

## Look at the screen the change lives on

- A change to a form field (a caption, a mark, a drop-down) is on the **detail screen**: press Edit on a row (or Add New) and read that screen. Checking the list page, the menu or the build proves nothing about it. Do this in every stack the change touches, and compare what you read with the database (marked captions against the NOT NULL columns, `sys.columns.is_nullable`).
- **Web stacks** (React, Angular and Blazor each with its own API port): click the navigation link, click the first Edit button, then read the open `dialog` (or the Blazor `.editor`) with `javascript_tool`: captions, the span that marks them, its computed colour, and `[...inputs].filter(e => e.required)`. A read-only look at the real database is fine; save nothing. Stop what you started by port when done.
- **WinUI3**: after each click check that the process is still alive (`Get-Process <name>`). A dialog that crashes on open leaves no process and no message; look at the `Application Error` event (`Microsoft.UI.Xaml.dll`, `0xc000027b`), then log `UnhandledException` (see the winui3 skill) to get the real text. A crash may be older than your change: build the last commit in a `git worktree` and run the same script on it before deciding whose it is.
- A regenerated sample builds even when a XAML resource lookup fails at run time. Only opening the dialog finds that.
- Save a WinUI3 screenshot with `CopyFromScreen` and read the PNG; it shows what UI Automation cannot (a red colour, a header inside a ComboBox). The method is in the `winui3` skill, `reference/driving-and-inspecting.md`.
