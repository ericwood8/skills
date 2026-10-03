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


## Database-level templates (one file for every table)

- A template that writes **one file for the whole database** (the `DbContext`, the API registration, an app shell) cannot take a `TableModel`. CodeGenNew gives it a `DatabaseModel` as the `Database` parameter (`<#@ parameter name="Database" type="CodeGenNew.Core.DatabaseModel" #>`), flagged by `DatabaseOnly=true` in its `.tt.config`; `TemplateRunner.RunAsync(path, databaseModel, project)` sets the session value, the CLI skips `-t`, the menu offers it on the database (not on a table: `AppliesTo` is false). The file name comes from `OutputName` with `{Table}` replaced by the project's context name.
- Put the rules that decide **which tables are listed** in `DatabaseModel` (Core), not in the template, and make them the same rules the per-table templates use (`API_Crud`: single int key, not name/active, `Project.NoRepository(...) != true`); otherwise the registration lists a table whose API was never generated. A lookup-shaped table gets a CRUD API but no search endpoint.
- Put the **provider choice in one generated static method** (`UseProvider(options, connectionString)`) that both the web registration and a desktop `OnConfiguring` call. The password never goes in `appsettings.json`: Npgsql reads `PGPASSWORD` itself; the Oracle MySQL driver does not read `MYSQL_PWD`, so the generated method adds it when the connection string has none.
- Declare the generated context `partial` (a context in a WinUI 3 project otherwise warns CsWinRT1028) so hand-written parts live in another file.
- The test that renders every shipped template must feed a `DatabaseOnly` template a `DatabaseModel`, or it fails with a missing-parameter error that looks like a template bug.
- Verify by swapping the generated files into each sample, building, and starting the API on a spare port (`--urls http://localhost:5090`: the `Urls` setting in `appsettings.json` otherwise wins) and calling a list, a lookup and a search route.

- **Never write `--` as a dash in template text.** Inside a generated XML / XAML / HTML comment it is illegal (the XAML compiler fails on the generated file, not on the template), and it is an easy habit in explanatory comments. Use ` - `, a colon or a period; for a ruler line use `=====`. Legitimate: `<!--` / `-->`, an SQL comment in an `SP_` template (`--` starts it) and C# `i--`. Name a command-line flag in words ("the project CLI parameter"), not as `--project`. A test that scans `Templates\*.tt` for any other `--` keeps it out.
- **A naming rule that changes generated names changes the JSON the shared front ends read.** Pascal-casing `require_customer_po` gives `RequireCustomerPo`, but the SQL Server copy says `RequireCustomerPO`, so a React/Angular front end written for one database shows `undefined` for the other. Keep an `Acronyms=PO,UPC,MSRP` project list (whole words only, any case) and take it from the other database's capital runs: `SELECT DISTINCT name FROM sys.columns WHERE name COLLATE Latin1_General_BIN LIKE '%[A-Z][A-Z]%'`.

- **Generate the list, keep the chrome.** For an app shell (menu, routes) the part that changes with every table is only the *list* of screens; the hamburger button and layout are not table-derived. Generate that list as its own file the hand-written shell imports (React `screens.tsx`, Angular `app.routes.ts`, a WinUI3 `partial class MainWindow` with `AddScreens()` / `ShowPage()`), so the hand-written file is edited once and never again. A WinUI3 `NavigationView` can take its items from code (`NavView.MenuItems.Add(...)`), which removes them from the XAML.
- **One source for names that two templates must agree on.** The route a master-detail screen links a child row to and the route the menu lists for that child are the same string; put the rule in Core (`ScreenNames`) and add a test that renders both templates and compares. Before writing a new generator over existing hand-written files, generate into a scratch folder and `diff` with the originals: here the React list was identical and the Angular file differed in two comment lines, which proved the rules matched before anything was replaced.
- **Warn, do not fail, for a soft consistency problem** (a child table with no screen): a comment at the top of the generated file keeps generation working for databases where it is legitimate (a junction table as a child); reserve `Error()` for input that cannot produce a correct file (a listed screen that is not a table).

- **Sorting from user input without dynamic SQL.** The caller sends a column *name*; the routine compares it with a fixed list of the table's own columns inside one `ORDER BY`, one `CASE WHEN @SortColumn = N'Col' AND @SortDescending = 0 THEN col END ASC, CASE WHEN ... = 1 THEN col END DESC` per column and direction (each `CASE` is its own sort key, so mixed column types are fine; a name that matches nothing makes every `CASE` NULL and the fixed default order that follows applies). Nothing from outside is ever concatenated into SQL, so there is no QUOTENAME / sp_executesql to get wrong; test it with a name like `x' OR 1=1 --`. A foreign key sorts by the parent's display column with a scalar subquery in the `THEN` (`(SELECT TOP 1 p.[Name] FROM [dbo].[Parent] AS p WHERE p.[Id] = t.[FkId])`), so alias the main table (`AS t`) and qualify every column in the sort expressions.
- **Adding parameters to a PostgreSQL function leaves the old one behind**: `CREATE OR REPLACE FUNCTION` with a different parameter list creates a second function, and a call that leaves defaulted parameters out then fails with "function is not unique". Generate `DROP FUNCTION IF EXISTS "s"."f"(<the old types>)` before the `CREATE`. A MySQL procedure has no defaults, so every caller passes the new parameters explicitly (a signature change is a regenerate of the callers too).
- **Fix the same name-vs-SQL-name slip wherever you touch it**: a PostgreSQL branch that wrote `Model.TableName` / `c.Name` into SQL (the generated names) works for a PascalCase database and breaks for `snake_case` read with `NamingStyle=Pascal`; SQL text takes `DbTableName` / `DbName`.
- **Shared front-end support code is a template too**: code every generated grid imports (the sort header, the saved-sort helpers) does not depend on a table, so write it with a whole-database template whose body is literal text after an `@@@FILE path@@@` marker (`TSX_GridSort`, `TS_GridSort`), and add it to each sample's `Regenerate.sh`; do not leave it as a hand-copied file with a comment saying "add this once".
- **A test column has to be one the screen would show**: a text column of 100 or more characters is a multi-line field and is left out of grids, so a test that sorts by it finds no header; and a column name with a quote breaks generated TypeScript and C# (keep it to the SQL test).

- **Calling a "clone" (or any routine that hands back a new key) from EF Core, per database**: SQL Server returns it in an `OUTPUT` parameter (`new SqlParameter("@NewId", SqlDbType.Int) { Direction = ParameterDirection.Output }`, `ExecuteSqlRawAsync("EXEC [s].[T_Clone] @CopyFromId = @CopyFromId, @NewId = @NewId OUTPUT, ...", ...)`, read `(int)p.Value!`; **named** arguments let optional parameters be left out); PostgreSQL returns it as the function's value (`SqlQueryRaw<int>("SELECT \"s\".\"T_Clone\"(\"CopyFromId\" => @a, \"pCode\" => @b) AS \"Value\"")`; `=>` named notation also skips defaults); MySQL has no OUTPUT-to-EF path and no default parameters, so the procedure ends with `SELECT v_new_key AS \`Value\`` and the caller passes **every** argument in the declared order. `SqlQueryRaw<T>` reads a scalar from a column called `Value`, so name it that in every database.
- **A copy must not trip a unique column**: give each unique text column a free value before calling the routine (`SuggestUnique<Column>`, "Good Standing" becomes "Good Standing2"), cut the source value short first so the suffix fits the column; offer the button only for tables whose unique columns are all text and whose key the database assigns. Check which unique columns the sample database really has before assuming (InvoiceSystem has none except the status tables' `Description`: clone a status row to exercise the path, then delete the copies).
- **A script that does not apply the SQL leaves the database behind the generated C#**: after a procedure signature changes (new sort parameters), the generated callers fail at run time with "too many arguments" while everything still builds. After regenerating, query the catalog for the new signature (`sys.parameters` for `@SortColumn`, `pg_proc`, `information_schema.PARAMETERS`) and apply with the script's `APPLY_SQL=1`; make every sample script generate and apply every routine the generated code calls (`_Search` and `_Clone`).
- **A grid that already holds all its rows sorts them on the client, by value, not by the text on screen.** A master dialog's child grids load the whole child table, so a header click re-orders the rows in the browser or the app (the paged grids sort in the database). Compare numbers as numbers, dates as dates, text without case, nothing first; give a column that shows a name in place of an id the **name** to sort by (the lookup is already loaded to show it). In WinUI3 the grid shows strings, so each row also carries its raw values (`Values`) and the sort uses those; sorting the formatted cells puts `$9.00` after `$10.00`. A child grid's column list is only known after its rows arrive, so make the saved-sort loader's column filter optional (web) or load the saved entry after the columns are known (WinUI3). Key the stored sort `<grid>.<child>` so each grid keeps its own.

## PostgreSQL enum columns and CHECK lists

- **Text will not go into an enum column.** EF Core sends a string as `text`; Postgres answers `column "status" is of type ticket_status but expression is of type text`. `HasColumnType` / `[Column(TypeName)]` do not fix it. Create an assignment cast (`CREATE CAST (text AS the_enum) WITH INOUT AS ASSIGNMENT`, guarded by a `pg_cast` lookup; make one from `character varying` too, because an entity with `[Column(TypeName = "varchar(n)")]` makes Npgsql send varchar and the cast lookup does not chain through text); a bad label is still refused. The `SP_EnumCasts` database-level template writes it.
- An enum has no `ILIKE` or text ordering of its own: cast the column to `::text` in search and sort.
- A CHECK list is stored as `= ANY ((ARRAY['A'::character varying, ...])::text[])`. Parse only that shape; OR/range checks must stay plain text. On an unbounded `text` column, set `maxLength` to the longest value or it is treated as long text.
- A unique enum column cannot be cloned (no suggested value); `CloneShape` must exclude it.
- A text in a template that contains the double hyphen of an SQL comment inside a C# string trips the dash test; build the marker with `new string('-', 2)`.
