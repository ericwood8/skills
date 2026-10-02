---
name: codegen-t4-templates
description: How to write and verify T4 (Mono.TextTemplating) code-generator templates in CodeGenNew that reproduce hand-written files — Mono T4 syntax quirks, error handling, multi-file output with @@@FILE markers, project-specific lists at the top of a template, the method of generating over existing code, diffing, and keeping as hand-maintained whatever cannot be reproduced, decoupling two features that got bundled under one flag by accident (e.g. pagination vs. searchability), syncing a template fix into a CLI-generated project's already-customized local template copy without losing project settings or hand-customizations, and using a test failure's own printed output as ground truth for fixing a wrong assertion instead of re-deriving expected text by hand. Use when adding or changing a CodeGenNew template (SP_, API_, CS_, TS_, WinUI3_), its .tt.config, the DefaultAssetSeeder list, or when deciding whether generated output may overwrite hand-written code.
---

# CodeGenNew templates that reproduce existing code

## Method that worked (SP, API, CS entity/enum/repo, TS model/service/component)

1. Read **all** the hand-written examples, not one. Note what varies: hand-picked names, extra members, deliberate relaxations.
2. Read the live database schema (read-only `sqlcmd`) and data for anything that becomes code (lookup rows -> enum members).
3. Write the template so its output for the simplest example matches the file **exactly** (diff should show only trailing whitespace, BOM, final newline). Put behaviour that is a property of the *project* — namespaces, context type, tables that are enums or have no entity — in variables at the top of the template, not in engine code.
4. Generate for **every** table, into a scratch copy or in place under git, then compare with the originals (`git diff -w --ignore-cr-at-eol`). Overwrite only what is a close approximation; **revert the rest and list it as hand-maintained**, with the reason (collection navigation, enum-typed property, custom route, a column the class deliberately relaxes).
5. Prove it: build the target solution, and where structure matters compare a machine dump (see the `efcore-aspnet-review` skill). Generated code for tables that have no screen/route must still be compiled — route or reference it in a scratch copy, because unreferenced files are not type-checked.
6. Never assume a bug is only in the sample: fix it in the template so all tables get it.

## Mono.TextTemplating quirks

- Signal a template error with `Error("...")` inside a statement block, then **wrap all output in the success branch** (or return no text); the run fails and the CLI prints the message without writing a file.
- Local functions and `var`/tuples work inside `<# #>` blocks; class-feature blocks are not needed. Build output with a `StringBuilder` and `Append('\n')` so line endings are exactly `\n` (a `Environment.NewLine` mixes CRLF into LF templates). Text like `GenericRepo<<#= entity #>>` is fine.
- Text produced by a template is written with `File.WriteAllTextAsync`: no BOM. Templates are LF; git normalizes to CRLF.
- A template gets only `Model` (a `TableModel`). Extra needs are switches in the template's `.tt.config`: `NeedsRowData` (enum members), `NeedsReferencedDisplayColumns` (foreign-key drop-downs), `RequiresPrimaryKey`, `TableOnly`, `OutputName` (a pattern with `{Table}`).
- `Templates` files are **embedded resources** in the CLI and app; after editing a `.tt` rebuild the CLI. The freshly built exe is in `bin\Debug\net10.0\`, not the older `win-x64\` folder (stale exe, "template not found"). New templates must be added to the list in `DefaultAssetSeeder`, named `Prefix_Name_v1.tt` (+ `.tt.config`); the group prefix (text before the first `_`) is the submenu and picks the default extension (`SP`->.sql, `API`/`CS`/`TS`->.cs/.cs/.ts).

## Multi-file output: `@@@FILE relative/path@@@`

A template that must write several files (an Angular component's css/html/spec/ts) or files in subfolders emits a marker line before each file; `GeneratedFiles` splits the output and writes each under the output folder (creating folders). A marker directly followed by another marker gives an empty file. Paths must be relative with no `..`, no drive. The CLI prints one `Wrote` line per file; point `-o` at the target project's root (Angular: `src\app`).

## Naming rules that matter

- The API prefix stripping is `E_` then `SY_`; route = lower-cased name + `s` (`E_DonateLeave` -> `/donateleaves`); the Angular dev proxy strips `/api`.
- JSON property names follow ASP.NET Core camel-casing (`FixCasing`): lower-case the first char; keep lower-casing while the *next* char is upper-case, stopping at the first char followed by a non-upper; stop after index 1 if that char is not upper. So `IsActive`->`isActive`, `SY_IsoCountry_Alpha3Code`->`sY_IsoCountry_Alpha3Code`, `E_TimeSheetId`->`e_TimeSheetId`. A TypeScript interface that says `e_timeSheetId` silently reads `undefined`.
- Navigation property name: the FK column without its trailing `Id` (`DonateFrom_EmployeeId`->`DonateFrom_Employee`); skip it if the name collides with a column.

## Tightening vs. loosening

Hand-written classes often *relax* the database: an entity property that is nullable although the column is NOT NULL, a navigation that is optional. A generator that follows the column makes such a property `required`, which makes the API refuse requests that omit it. Treat "generated is stricter than the original" as a reason to keep the file hand-maintained; "generated is looser" (navigation `required` -> nullable) is acceptable but must be reported.

## Decoupling two features that got bundled together in an early version

A generated screen/API can end up with two genuinely independent features gated by the *same* condition
just because the first table tested happened to need both together — e.g. a `canPage` flag that also
silently controlled whether the search bar showed up, because the first tables generated all had a
searchable column. Once a table with **no** searchable column exists (or the user explicitly says so —
here: "System wide. No pagination bar on Customer Monthly Summaries" — pagination should exist regardless
of searchability), the bundling becomes a bug, not a simplification. Fix by introducing the second flag
explicitly and re-deriving the first from only what it actually depends on:
```csharp
bool canSearch = searchFields.Count > 0;   // gates the search bar UI only
// pagination (PaginationBar, PageNumber/TotalPages, SearchAsync call) is now unconditional
```
When doing this, **delete the old single-flag branches entirely** rather than leaving them as unreachable
dead code (e.g. a `GetAll()`-fallback branch that only fired when the old combined flag was false) — this
touches more lines across more templates than a minimal patch, but leaving dead branches violates "no dead
code" and confuses the next reader into thinking the fallback path still matters.
This exact split had to be threaded through every layer that independently implements search-or-not: the
stored procedure (`SP_Search.tt` — WHERE clause becomes optional, not refused for zero filters), the API
endpoint (`API_Search.tt`), the repository method (`CS_Repo.tt`), and both UI stacks per screen type
(`TS_Component.tt`/`TSX_Page.tt` for web, `WinUI3_MasterScreen.tt` for desktop) — miss one layer and that
layer alone keeps the old bundled behavior.

## Sync a template fix into the CLI's already-generated project

`Templates/*.tt` in the CLI's own `bin/Debug/net10.0/Templates/` folder is a **separate copy** from the
repo source under `Templates/` at the repo root — editing the repo source and rebuilding refreshes this
copy from the shipped embedded resource, but any project-specific customization already applied at the top
of that local copy (namespaces, `contextType`, table-is-enum lists, etc. — the "PROJECT SETTINGS" block
described above) gets overwritten in the process, since it's the same file. Workflow when fixing a template
that an existing generated project (e.g. a WinUI3 app already scaffolded from an earlier CLI run) depends
on:
1. Fix the bug in the repo source `Templates/<Name>.tt`, verify via the test suite (`dotnet test`).
2. Rebuild the CLI so the local copy refreshes from the fixed source.
3. Re-apply that specific project's settings into the CLI's local copy of the template (the same edits made
   when the project was first scaffolded — a plain overwrite/rebuild wipes them every time).
4. Regenerate the affected files for that project via the CLI.
5. Re-apply any **hand-customizations** made directly to the generated files after the fact (wiring a
   generated dialog's button to a hand-added drill-down screen, an added `Entity` property on a generated
   row class) — regeneration overwrites the file from scratch, so anything not expressible in the template
   itself has to be reapplied by hand every time that file regenerates.
6. Rebuild the target project and confirm 0 errors before calling the fix done.

## When a test assertion is wrong, use the actual failure output as ground truth, not a re-derivation

When a template's raw multi-line text output doesn't match a hand-written string assertion (e.g. asserting
`"EXEC [dbo].[Foo_SearchCount]\", countParameters)"` as one contiguous string when the template actually
emits the closing paren on a separate line from the SQL string), don't guess the correct expected string by
re-reading the template and mentally re-rendering it — copy the **exact** generated text straight out of the
test failure's own printed "Expected to find... in:" block and split/quote the assertion to match that
literally. A string-builder-based template (one `.Append(...)` call assembling one output line) and a raw
`<# #>`-templated one (literal template text with interpolations, spanning exactly the lines written in the
`.tt` file) can legitimately format the "same" logical output differently — don't assume two templates that
produce conceptually equivalent code will pass the identical assertion string.

## Things a generator cannot know (say so in the template header)

Collection navigation names, reverse navigations already listed by the parent (adding both sides makes JSON serialization loop), enum-typed properties, validation attributes, hand-picked sort columns, screens with detail grids or dependent drop-downs, and registration lines (DbSet, routes, sidebar).


## Regenerating a sample project from the templates: the workflow that works

- **One script per sample** (`docs\Regenerate.sh` beside the project): rebuild the CLI, **delete the CLI's `Templates` folder** (it seeds embedded templates once and never overwrites; a stale
  folder runs old templates and looks like a template bug), copy the project's `.config` into the CLI's `Projects` folder, generate every template/table into a temp folder, copy into the
  project, print only `diff -rq --strip-trailing-cr` against a backup taken first, then build and test. Keep the table list per template at the top; a **refusal** (a template that
  refuses a table) is printed, not hidden.
- **Schema changes need the database side too.** Generated search stored procedures list their columns, so after a column is added or renamed re-create them (`APPLY_SQL=1`) or the
  app fails at run time with a missing-column error that the build cannot see.
- **A lookup table is not always an enum.** With the project's name and shape rules a small `*Status` table is derived as an enum (no entity, repository or API), so a screen shows its
  foreign key as a number. Set `EnumTables=none` in the project file when the table's description should be shown.
- **Template files are CRLF in a Windows checkout**, so generated text has `\r\n`; normalise (`Replace("\r\n", "\n")`) in test helpers that assert multi-line text.
- **The generator and the front end must agree on names.** CodeGenNew's `Pluralizer` builds the TypeScript routes; any hand-written API base class has to implement the same rule
  (a trailing bare `s` stays, consonant + `y` becomes `ies`) or pages 404.
- Anything a template needs to know about a **child** table (money columns, long text, captions) is available at generation time from `ChildForeignKeyModel.ReferencingTableColumns`; prefer
  that to guessing from names at run time.


## Rules that must be the same on every platform belong in `CodeGenNew.Core`, not copied into each template

Examples now in Core and called from the WinUI3, Angular and React templates: `FormPages.For(columns)` (Main / Billing & Shipping / Notes tabs), `GridColumns.ForGrid(columns)` (no long text, at most 18), `GridCaption.For`, `NumericClassifier.DecimalRange`, `DisplayColumnSelector`. A template then asks one question and all stacks answer the same way; a unit test of the Core method
covers every platform. Per-template edits of six near-identical templates drifted (one got the fix, another did not) before this.

## Editing a large `.tt` with a script

Do multi-line edits with a small Node script written to a **file** (Write tool) that does exact-string `replace` and **throws when the match count is not exactly 1**; heredocs of a few hundred lines get rejected by the shell, and `\r\n` typed in a heredoc turned into a real line break inside generated C# test strings
(fix with a regex pass over the test file). Use `s.replace(a, () => b)` so `$` in the replacement is not interpreted.

## Regenerating the three samples

`C:\InvoiceSystem` (WinUI3, `APPLY_SQL=1` after a column change), `C:\InvoiceSystemReact` and `C:\InvoiceSystemAngular` each have `docs\Regenerate.sh`; run one per template change and read the last lines. **Stop the sample's API first** (a running API locks its exe: `MSB3021`), and do not re-run a script "just to be sure" while the servers are up.
Order after a schema change: WinUI3 with `APPLY_SQL=1`, then React, then Angular. The Angular dev server and Vite pick up regenerated files by themselves; reload the page.
Making a new web sample: `ng new frontend --routing --style=css --ssr=false --zoneless=false --skip-git --ai-config=none --defaults --test-runner=vitest --skip-install`, add `@angular/material`, `@angular/cdk`, `@angular/animations`, then the hand-written files listed in that sample's `Regeneration.md`.

## One generator, several databases (SQL Server, PostgreSQL, MySQL)

- **One shape.** `ISchemaProvider` / `SchemaProviderBase` do everything that is not a catalog query (display columns, lookup shape, special-logic rules, child tables); a provider implements only the catalog SQL and the type mapping (`SqlServerSchemaProvider`, `PostgresSchemaProvider`, `MySqlSchemaProvider`). Map every database's types onto the SQL Server vocabulary the generators classify by (`int`, `bit`, `datetime2`, `varchar(-1)` for text) and keep the database's own spelling in `SqlTypeDeclaration`. `TableModel.Dialect` tells a template which SQL to write.
- **Names: generated vs real.** `NamingStyle=Pascal` (project setting / `--naming`) converts `customer_item` to `CustomerItem` at the schema reader; the SQL keeps the real names (`ColumnModel.DbName`, `TableModel.DbTableName`, `ForeignKeyModel.ReferencedDbTable/Columns/DisplayDbColumns`). Rule of thumb: **a C#/TypeScript identifier uses `Name`; anything inside SQL text or a `[Table]` / `[Column]` attribute uses `DbName`.** EF allows one `[Column]` per property, so write one attribute carrying both the real name and the type. Classification rules (audit columns, currency names) read the generated name.
- **Dialect-specific SQL lives in a Core class, not in the .tt.** `PostgresProcedures`, `MySqlProcedures`, `SearchCall`, `PostgresLiteral`, `MySqlLiteral` are plain C# (unit-testable, no template compile). The `SP_*.tt` file starts with `if (Model.Dialect == SqlDialect.X) { Write(...); return GenerationEnvironment.ToString(); }` and the T-SQL text below it stays as it was. A template that cannot work for a dialect says `SqlServerOnly=true` in its .tt.config (refused with a reason; hidden in the app menu).
- **Dialect facts that bit.** SQL Server `EXEC` of a procedure is not composable; PostgreSQL functions that `RETURN SETOF table` are (`SELECT * FROM f(...)`), and a scalar count needs `AS "Value"` for `SqlQueryRaw<int>`; PostgreSQL wants parameters without a default first, so call by name; `information_schema` booleans can be NULL (`COALESCE`), a bug only a live test finds; Npgsql maps `DateTime` to UTC-only `timestamptz` unless the column type is `timestamp`; MySQL has no default parameter values, no `CREATE OR REPLACE PROCEDURE` (`DROP ... IF EXISTS` + `DELIMITER $$`), no `RETURNING` (`LAST_INSERT_ID()`), treats a backslash in a string literal as an escape, stores table names in lower case on Windows (`lower_case_table_names=1`), and its `ROW_COUNT()` is 0 when an update changes nothing; Pomelo's MySQL EF provider lagged EF Core (use Oracle's `MySql.EntityFrameworkCore` / `UseMySQL` until it catches up).
- **Test live, with credentials from the environment.** Unit tests cover type/default mapping, connection strings and the text of each procedure; an integration test class runs only when `CODEGENNEW_<DB>_HOST/_DATABASE/_USER/_PASSWORD` are set (otherwise `Assert.Inconclusive`). Passwords go through `PGPASSWORD` / `MYSQL_PWD`, never into a file of a repo.
- **A sample per database** (`C:\InvoiceSystemPg`, `C:\InvoiceSystemMySql`): an API project and a WinUI3 app with their own `Regenerate.sh` / `RegenerateWinUI3.sh`; the React and Angular front ends reuse the API unchanged (same routes and JSON), so checking a new database in the browser is just starting its API on port 5080 / 5081.
