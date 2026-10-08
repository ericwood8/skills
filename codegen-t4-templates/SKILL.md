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

## Multi-file output with FILE markers  ```@@@FILE Models/Customer.cs@@@```

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

## Whole-project generation: the plan lives in the template configs

- A template joins "generate everything for a project" with config keys, not a list in a script: `Stacks=`, `PlanTables=` (which tables: Entity, Api, Search, Screen, ScreenForm, ScreenDetailMaster, Junction, Enum), `OutputRoot=` and `OutputFolder[.<Stack>]=`; `InPlan=false` makes it opt-in (`PlanAlso` in the project); `Dialects=` limits it to one database. A new template that is missing these is silently left out of `codegen generate`.
- **Compile each template once per run** (`TemplateCache`): Mono.TextTemplating's compile is the cost (about a second a run), and the compiled class re-reads `Model` / `Database` / `Project` from the session each time. Re-set the session values and clear `Errors` between runs; a template's `Error(...)` then fails that run only.
- A no-database template (`NoDatabase=true`, only the `Project` parameter) writes its own paths with `@@@FILE`; the essentials groups (App, MainWindow, styles, Program.cs ...) are such templates. Compare their output with a working sample's hand-written files, normalising line endings, before trusting them.
- Write only when the text differs after normalising `\r\n`: a CRLF checkout then counts as unchanged, and "what changed" is a real list.

## Adding a template: the checklist

A new template touches more than the two files; each of these was missed once and caught by a test or a run:

1. `Templates/<Group>_<Name>_v1.tt` and `.tt.config`. The group prefix picks the menu and the default extension; a **new prefix** needs an entry in `TemplateInfo.ExtensionByGroup` (`FS` -> `fs`).
2. `DefaultAssetSeeder.DefaultTemplateFileNames` (both files). Templates are embedded and seeded once; a template missing from the list is not shipped.
3. The group counts in `TemplateEngineTests.The_shipped_templates_are_all_offered_with_their_groups`.
4. A `Docs/TemplateNotes/<Name>_v1.md`, the README table row and the "N templates ship" count, a settings hint (`ProjectSettingsDialogViewModel.Hints`) and the key in `ProjectSettings.Keys` for any new project setting.
5. A rendering test, and for generated **code** a real compile: generate for several tables of a sample database into a scratch project (add the sample's entities and `Microsoft.EntityFrameworkCore` if they need it) and run a small round trip. Output that only looks right has failed to compile before (`typeof(string?)`, a missing `using`).
6. For SQL: apply the script to a scratch copy of the database.

Shapes (pick by what the template needs, not by its name):
- per table (`Model`): the default; `RequiresPrimaryKey`, `TableOnly`, `OutputName={Table}Dto.cs`.
- `DatabaseOnly=true` (`Database`: every table): one file for the whole database, written with an `@@@FILE name@@@` marker.
- `NoDatabase=true` (only `Project`): run by hand or as an essentials group; **a whole-project plan never includes a NoDatabase template** unless it has an `EssentialsGroup`.
- `InPlan=false` + the project's `PlanAlso`: in a plan only when asked. `Dialects=SqlServer` skips it quietly for another database.
- `SqlServerOnly=true` is forbidden by a test (every shipped template runs on every database): use `Dialects=` and, for a per-table template someone may run by hand on another database, an `Error(...)` on `Model.Dialect`.

T4 traps met while doing this:
- A compile error in a `.tt` comes back in `TemplateResult.Errors`; put them in the assertion message (`string.Join(" | ", result.Errors)`), a bare `Assert.IsTrue(result.Success)` shows nothing. Usual cause: a missing `<#@ import namespace="System.Data" #>` (for `SqlDbType`) or `System.Collections.Generic` (for `List<>`).
- The dash test: no `--` anywhere in a `.tt` except an SQL comment at the start of a line of an `SP_` template. Inside a C# string in the template build the marker once: `string Rem = new string((char)45, 2);` and write `Line(Rem + " text")`.
- `ColumnModel.MaxLength` of an `nvarchar` is in **bytes**: halve it for a character limit (CS_Entity does).
- `{x}` inside an interpolated string is fine, but `global::Ns.Type` inside `{ }` of an interpolated string is read as a format specifier (`::`): use a variable.
- Names that are reserved words come from the schema reader (`IsCSharpReservedWordName`); `Sample.Column` in the tests sets it from the name, so a test column called `class` exercises the `@class` path.

## Generated documents and specs: how to check them

A template that writes a document (OpenAPI, Markdown, Mermaid) is easy to get subtly wrong, and the usual rendering test only compares text. Checks that paid off:
- **OpenAPI:** read the output back with `Microsoft.OpenApi.Readers` (`new OpenApiStringReader().Read(yaml, out var diagnostic)`; assert `diagnostic.Errors` is empty and look at `Paths` / `Components.Schemas`), then **probe the running sample API**: every documented GET answers 200, `PUT /{id}` with a wrong body answers 400 (the route exists; a missing route is 404, and so is a missing row, so a GET of a missing id proves nothing), and the schema property names equal the keys in a real response. Routes written "by rule" drift from the API unless something checks them.
- **Mermaid:** `npm i mermaid jsdom`, put `window` / `document` from jsdom on `globalThis` and call `await mermaid.parse(text)`; also parse a deliberately broken line, to know the check can fail. The browser pane renders a local file as a static snapshot (no scripts), so it cannot do this.
- **Markdown tables:** a pipe inside a cell breaks the row (escape it); the separator row and Mermaid's relationship arrows need hyphens, which the double-hyphen rule forbids in template text: build them with `new string((char)45, n)`.
- **FluentValidation and other libraries:** compile the generated classes in a scratch project that references the sample's real entities and the library. Overload inference failed on `InclusiveBetween(0, 100)` for a `short?`; only a compile showed it.
- A word that already ends in `s` is not pluralised (`CustomerStatus` is `/api/customerstatus`): take plurals from `Pluralizer`, never guess them in a test.
- Use `$(cygpath -u "$TEMP")` for paths in Git Bash: an unquoted `$TEMP/dir/*.cs` with backslashes in the value silently matches nothing.

## Shared helpers and switches (2026-10)

- **Do not copy a type map or a caption rule into a template.** `CodeGenNew.Core` has them: `ColumnTypes` (`CSharpName`, `CSharpBase` / `CSharp`, `FSharpBase` / `FSharp` / `FSharpParser`, `TypeScript`, `JsonSchema`, `CharacterLength`), `Labels.Words` and `JsonNames.Camel`; `TableModel.HasCrudApi` / `HasSearchApi` say whether a table has an API / a search endpoint. A new language or rule is added there with a test over every `SqlDbType`, and the templates call it. Moving a copy out of an existing template is safe when **every sample dry-runs unchanged** (`codegen generate ... --dry-run` for each sample's project: 0 created, 0 updated); a script that does the seven samples is the regression check.
- **Optional generated extras are switches, not template lists.** A template with `InPlan=false` joins a whole-project run through the project's `PlanAlso`, or through a flag that `ProjectSettings.ImpliedPlanTemplates` maps to it (`ApiDocs`, `ApiHttp`, `ApiFakers`, `ProjectDocs`, `ApiValidation`). A flag may also change an essentials template (`API_EssentialProgram` adds the package reference, the registration and the extra files) or a template's body (`API_Crud` adds the validation filter): keep every such branch **off by default** so the samples do not change, and test both states.
- **What the database says about itself** (a column or table comment) is read by each provider into `ColumnModel.Description` / `TableModel.Description`: SQL Server `sys.extended_properties` (`MS_Description`, `minor_id = 0` for the table), PostgreSQL `col_description` / `obj_description` over `to_regclass(format('%I.%I', ...))`, MySQL `COLUMN_COMMENT` / `TABLE_COMMENT`. Templates show it only when present, so a database with no comments produces the same output as before.
- **A generated project is the best test of the extras:** build it, run it, and use the new files (`.http` bodies, fakers posted to the API, validators refusing a bad row, the Swagger page). A scratch console project with a `ProjectReference` to the generated Web project can call the generated classes directly.

## Small templates round (2026-10): what to reuse and how to check

- **One fact list for validators.** `ColumnModel.RuleOf(project)` (Core, `ColumnRules`) says what the schema states about a column (required, length, choices, email / phone / url shape, ranges). `CS_Validator`, `TSX_Schema` (zod) and `TS_Validators` (Angular) all read it; a new validator-like template must too, not re-derive the rules from names. A type map for a new language goes in `ColumnTypes` with a data-row test over every `SqlDbType`.
- **New input from the database** (for example `TableModel.Indexes`): add a `protected virtual` read to `SchemaProviderBase` (empty by default), override it in the three providers with a catalog query, group rows with a shared helper, and add a live test per database that builds a scratch table and drops it (`SchemaIndexTests` is the pattern). Leave out what does not apply to every row (filtered, partial, expression indexes).
- **A T4 template that uses `List<>` or `IEnumerable<>` must import `System.Collections.Generic`** (the compile error is "type List<> could not be found" and the line points into a temp file). `Error(...)` in a plan is a failure of the run, not a refusal: for a table with nothing to say, write an empty result instead (refusals come from the template's `.tt.config`).
- **A test helper that renders a template should `Assert.Fail` with the errors joined on one line** (`ReplaceLineEndings(" ")`); `Assert.IsTrue(cond, $"...")` printed an empty message for a multi-line error.
- **Prove the output with the real tool**, not only with string asserts: `tsc --strict` over every generated `.ts` file (install zod and `@angular/forms` in a scratch folder), `Grpc.Tools` (the real `protoc`) for `.proto`, a scratch project with the package for Mapperly or FluentValidation, and the real database for generated SQL (SQL Server inside `BEGIN TRAN ... ROLLBACK`, PostgreSQL on `CREATE DATABASE x TEMPLATE y`, MySQL by applying and dropping).
- **Editing from Git Bash:** do not patch C# or T4 with perl one-liners that carry `\n`, `\r` or quotes (they inject real newlines); use the Edit tool for any change that has an escape in it.

## Adding a database and an access mode (2026-10)

- **A new database needs:** a `SqlDialect` and `DatabaseProvider` value, a connection factory in `CodeGenNew.Connections`, a `SchemaProviderBase` subclass (the abstract members: `ReadColumnsAsync`, `ReadForeignKeyRowsAsync`, `ReadRowCountAsync`, `ReadRowsAsync`, `ListTablesAsync`, `ListColumnSummariesAsync`; override `ReadIndexRowsAsync` and `ReadTableDescriptionAsync` when the catalog has them), `SchemaProviderFactory`, the CLI (`--provider`, default schema, what `-S`/`-U` mean), the desktop dialog and the menu's dialect mapping, `EfProviderInfo`, the essentials (package, connection string) and `DialectInfo`. Types go in the SQL Server vocabulary so no template learns the new type names. Tests that build a real database in a temp file need no installed server and always run.
- **A template for some databases says `Dialects=` in its config** (the plan skips it silently, the menu hides it, a single run is refused with the reason); **a template that belongs to one access mode says `AccessMode=Routines|Ef`** (the plan skips the other mode's, unless `PlanAlso` names it). Check every `Dialect ==` ternary that falls back to SQL Server: a new dialect lands in that branch by default.
- **Put what a stack other than EF would also need in Core, not in a template:** `SearchPlan` (what a search does), `DialectInfo` (quoting, paging, placeholders, contains, date buckets), `EntityNavigations`, `CloneShape`. A template then renders; it does not decide.
- **Generating code lines from Core or a template:** a T4 helper `void Emit(List<string> lines)` that calls `Write(line)` and `Write(((char)10).ToString())` avoids escape trouble; do not write `"\n"` through perl or a heredoc.
- **Prove a new mode or database end to end, cheaply:** a scratch database file, `codegen generate ... --essentials --build`, run the API with the connection string in the environment, `curl` the routes (search with a filter and a sort, clone, link and unlink), then `Test-ApiCrud.ps1` and `Test-OpenApiRoutes.ps1` from the PowerShell tool (a path built with `cygpath` and passed from Git Bash to `powershell -File` failed to start the API; the PowerShell tool worked). Then dry-run the seven samples: a mode that is off by default must change nothing.
- **To look at what the schema reader decided about a column,** write a throwaway console project with `ProjectReference`s to Core, Connections and SchemaIntrospection and print the model's flags; a diagnostic `.tt` run through the CLI gave only a stack trace.


## Adding a project setting: every place it has to appear

A new `ProjectSettings` key is not finished when the property works. Also:
- the `Keys` list, a one-line hint in `ProjectSettingsHints` (explain the effect and what blank means; do not start the hint with the value to type), and `Docs/Reference.md` (a test checks all three);
- a tab in `ProjectSettingGroups` (at most 20 settings per tab, a test enforces it and that every key is on exactly one tab);
- the right control: `ProjectSettingsHints.BooleanKeys` for true/false (check box), `ProjectSettingChoices` for a fixed list (radio buttons, drop-down, or multi-select) and its `Numbers` table for whole numbers with a range (number box). A test keeps the choice lists equal to the code that parses them (for example the stack list equals `ProjectPlan.KnownStacks`);
- a flag that adds templates (`ImpliedPlanTemplates`) must say in its hint that the project has to be regenerated, and the stacks it needs: a user who ticks it and opens the app sees nothing until Generate All has run with the right stacks ticked. Blank `Stacks` generates nothing (the dialog demands a tick), so say that where the setting is shown.
- an optional feature that is off by default is a deliberate choice (it adds files to existing projects and may add an unauthenticated endpoint); say why in the answer when asked, and make it as easy to turn on as to find.

## A template that only fits some tables: derive it from the table, not from a setting (2026-10)

The audit trail (`SP_AuditTable`) first needed a project setting (`AuditTables=Customer,Item`) to say which tables it was for. That was wrong: a user who ran it on a table not in the list got a refusal about a setting they had never heard of. What the table itself shows is better.
- **Put the rule in Core, once.** `AuditTableShape` (a creation column and a change column) is used by `TableModel.IsAuditTable` and by `AuditColumnClassifier`, so the model and the menu cannot disagree. A test builds tables with and without the columns.
- **Menu filtering uses the cheap `TableSummary`, not the full model.** A new "requires" key needs: the property and its parse in `TemplateConfig`, a reason in `TemplateConfig.Refuse` (single run), a parameter on `TemplateInfo.AppliesTo` (menu), the flag on `TableSummary` computed in **every** provider's bulk catalog query (SQL Server, PostgreSQL, MySQL, SQLite), the call in `MainViewModel.GetApplicableTemplates`, and a row in `Docs/Reference.md` (a test checks every config key is documented).
- **Keep it out of the default plan** with `InPlan=false` when it adds tables or triggers to a database; `PlanAlso` turns it on, and `PlanTableSet` picks the tables.
- **A template that writes a second table must refuse clashes up front** (a column already called `AuditId`, a table that is already system-versioned) with a reason, never fail half way.
- When removing a setting, remove it from the keys list, the tab list, the hint, `ImpliedPlanTemplates`, `Reference.md`, the template header, the notes, the README and the specs; the guard tests find the ones you forget.
- Explaining a history design to a user: walk one row through create, change, review, delete, review. The honest gaps (no insert row, only the before-image, `AuditUser` is the database login and not the application user, not tamper-proof, a delete does not touch `ModifiedBy`) are what they ask about next, so say them first.

## Adding a stack (Blazor, 2026-10)

- **Plumbing a stack needs** (found by following Rust): `ProjectPlan.KnownStacks`, `ProjectSettingChoices.Stacks`, the keys `Output<Stack>` / `Build<Stack>` / `Test<Stack>` in `ProjectSettings.Keys` (and `OutputFolderOf`), a hint for each in `ProjectSettingsHints`, the tab lists in `ProjectSettingGroups`, a row for each in `Docs/Reference.md`, `EssentialsCatalog.Stacks` and the Essentials menu item in `MainWindow.xaml`, `ProjectBuilder.DefaultCommand`, `TemplateInfo.ExtensionByGroup`, `DefaultAssetSeeder`, the group count in `TemplateEngineTests`, the CLI and settings text that list the stacks. A front end the API is called from needs its origin in `API_EssentialProgram`'s CORS list.
- **Which tables get a file** is one rule in Core (`BlazorNames.HasClient` / `Refusal`) used by the per-table templates and by the whole-database template that registers them, so a page never asks for a client that was not written.
- **Razor traps in generated markup:** `@page` is a directive, so a field named `page` fails with RZ2005 ("must appear at the start of the line"); `Name@Arrow(x)` is read as an e-mail address and printed literally (put a space before `@`); an attribute with a C# string inside takes single quotes (`@onclick='() => Sort("Name")'`); a header arrow written as `▲` in a C# string needs `\u25B2` in the template line.
- **Writing Razor from a template:** build lines with a small `L()` helper and replace tokens (`§T`) instead of nesting `{{ }}` in interpolated strings. Bash heredocs that contain a lone quote can fail with "unexpected EOF"; write such files with the Write tool.
- **Prove it end to end with the dev servers:** generate the API and the front end from the PostgreSQL sample (`--project` with a config in a scratch folder, `-P` on the command line for one run), build both, run both, drive the app in the browser pane (JavaScript that sets values and dispatches `input` and `change`, `window.confirm = () => true` for a delete), then delete what the test added. Stop a server by its port (`Get-NetTCPConnection -LocalPort`), not by a command-line match, which kills the shell that asked.

## A Python stack beside the .NET ones (2026-10)

- **Same wire contract, not same code.** The Python API keeps the routes, JSON names (camelCase), search answer (`items, page, pageSize, totalCount, totalPages`) and error shapes of the .NET API, so every front end runs against it unchanged. That is the test of a new back end: point the existing React/Blazor app at it.
- **Keep the type map and naming rules in Core (`PythonTable`, `PythonNames`)**, not in the templates: reserved words (`class`, `from`, `id` shadowing is fine, `type` is not), snake_case, unsupported types left out with a comment listing them. Templates stay thin and tests can call the map directly.
- **FastAPI answers 422 for a bad body**; to match ASP.NET register handlers for `RequestValidationError` and your own `ApiValidationError` that write `{ type, title, status, errors: { Property: [messages] } }` as `application/problem+json` with status 400. Pydantic writes a `Decimal` as a string: annotate it with `PlainSerializer(float, when_used="json")`.
- **SQLAlchemy 2.0 style** (`Mapped[...]`, `mapped_column`, `select()`), `sessionmaker(expire_on_commit=False)` so a saved row can be returned after the commit; sync routes are enough and simpler.
- **Never write the password**: read `PGPASSWORD`, `MYSQL_PWD` or `DB_PASSWORD` in `db.py`, with `DATABASE_URL` as an override; `.env.example` says where to set it.
- **A template line may not hold two hyphens in a row** (the dash guard test), which bites a CLI flag in generated text (`uvicorn app.main:app --port 5080`): build it with `new string((char)45, 2) + "port"`.
- **A flag that implies a template** (`ApiValidation` now adds `PY_Validate` as well as `CS_Validator`) changes the list `ImpliedPlanTemplates` returns, so the test that lists it must change too; grep tests for the old list when you add an implied template.
- **Prove it live in a throwaway venv** (`python -m venv`, `pip install -r requirements.txt`, about 70 MB): ask before installing anything; run uvicorn against the dev database with the password in the environment of that one command; drive get/search/create/update/clone/delete and the error cases from a small stdlib `urllib` script; then run the existing front end against it. Test deletes on a row with dependants too (expect 400, nothing deleted) and put back every row you touched.
- A test can byte-compile the generated package with `python -m compileall -q app` and skip when Python is not installed.
- A new project setting also needs a tab in `ProjectSettingGroups` (a key no group lists lands on the first tab); a test fails until the key is in `Keys`, the hints, a group and `Docs/Reference.md`. A switch that adds packages or files to generated output defaults to off.
- Template lines must not contain a double hyphen, even inside C# or XML text; reword it.
