---
name: codegen-t4-templates
description: Use when adding or changing a CodeGenNew T4 (Mono.TextTemplating) template (SP_, API_, CS_, TS_, TSX_, WinUI3_, BLZ_, PY_, RS_), its .tt.config, DefaultAssetSeeder or ProjectSettings; when adding a setting, stack, database or access mode; when regenerating a sample or an already generated project; or when deciding whether generated output may overwrite hand-written code. Trigger words - T4, .tt, @@@FILE, essentials, Core helper, Regenerate.sh, stale template, ListProtocol, ImpliedPlanTemplates.
---

# CodeGenNew templates

This file is the short core. Find your task in the table, read that one file in `reference/`, and read a second only when the first points to it. Each reference file starts with a "Read this when" line.

## Pick the file

| You are about to... | Read |
|---|---|
| Reproduce hand-written code, or decide whether generated code may replace it | [reproducing-code](reference/reproducing-code.md) |
| Fix a template that will not compile, has wrong line endings, quotes or `--` | [t4-traps](reference/t4-traps.md) |
| Emit several files, or get a route, JSON name, navigation name or SQL name right | [names-and-output](reference/names-and-output.md) |
| Put a decision, name list, sentence or model rule where several stacks share it | [core-rules](reference/core-rules.md) |
| Write a status code, exit code, database error number or an API error answer | [named-numbers-and-problems](reference/named-numbers-and-problems.md) |
| Move repeated per-table code onto a base class, or change a wire contract (`ListProtocol`) | [essentials-and-bases](reference/essentials-and-bases.md) |
| Add or change a template, `.tt.config` key, project setting, or a template that fits only some tables | [plan-and-new-templates](reference/plan-and-new-templates.md) |
| Add a stack, a database, an access mode or a new kind of schema input | [new-stack-or-database](reference/new-stack-or-database.md) |
| Write SQL, call a routine, or handle one database differently | [sql-dialects](reference/sql-dialects.md) |
| Change paging, search, sorting or grid columns of a screen | [paging-and-grids](reference/paging-and-grids.md) |
| Generate OpenAPI, Mermaid, Markdown, validators, or test generated text | [documents-and-checks](reference/documents-and-checks.md) |
| Regenerate the samples and prove nothing else changed | [regenerate-and-prove](reference/regenerate-and-prove.md) |
| A generated project fails after new templates, or the first Generate All misbehaves | [existing-projects](reference/existing-projects.md) |
| Touch a user table, password hash column or sign-in | [user-table-and-sign-in](reference/user-table-and-sign-in.md) |
| Touch required marks, a WinUI3 dialog, a single-row table, a generated key, or walk pages in a browser | [screens-and-dialogs](reference/screens-and-dialogs.md) |
| Generation is slow, the cache is stale, or a sample builds with warnings | [speed-and-cache](reference/speed-and-cache.md) |

## Rules that apply to every change

1. **A template renders; it does not decide.** A rule that every stack must share goes in `CodeGenNew.Core` with a test, and each template calls it ([core-rules](reference/core-rules.md)).
2. **Fix the template, not the sample.** A bug fixed only in one generated project comes back at the next regeneration, and every table must get the fix.
3. **Stale template first.** The CLI seeds its `Templates` folder once and never overwrites it: delete `bin\Debug\net10.0\Templates` before a run, and look for `*.new` files beside a template that "still fails" ([t4-traps](reference/t4-traps.md), [existing-projects](reference/existing-projects.md)).
4. **Never write two hyphens in template text**, and edit templates with the Edit tool or an exact-one-match script, never a shell string ([t4-traps](reference/t4-traps.md); see also the `git-bash-editing` skill).
5. **Essentials are written once and kept.** Do not change what an essentials file must contain to make a page template work; put the new UI in the page template ([essentials-and-bases](reference/essentials-and-bases.md)).
6. **Keep default output unchanged** when a setting is off, and prove it with a diff against the last commit. Optional extras default off and are tested in both states.
7. **Prove with the real tool**, not only the template tests: build the generated code, run the API, click through the screen. A green unit test of template text misses real defects ([regenerate-and-prove](reference/regenerate-and-prove.md), and the `verify-in-running-app` skill).
8. **Report what you could not generate or prove** (columns left out, files kept hand-maintained, a stack with no sample) instead of hiding it.

## Related skills

`odata-generated-grids` (grid columns, `$select`, owner scope), `verify-in-running-app` (scratch database and click-through), `winui3` (WinUI3 patterns), `angular-crud-gotchas`, `react-crud-gotchas`, `efcore-aspnet-review`, `mssql`, `mysql`, `postgres`, `git-bash-editing`.
