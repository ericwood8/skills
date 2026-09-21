---
name: codegen-t4-templates
description: How to write and verify T4 (Mono.TextTemplating) code-generator templates in CodeGenNew that reproduce hand-written files — Mono T4 syntax quirks, error handling, multi-file output with @@@FILE markers, project-specific lists at the top of a template, and the method of generating over existing code, diffing, and keeping as hand-maintained whatever cannot be reproduced. Use when adding or changing a CodeGenNew template (SP_, API_, CS_, TS_), its .tt.config, the DefaultAssetSeeder list, or when deciding whether generated output may overwrite hand-written code.
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

## Things a generator cannot know (say so in the template header)

Collection navigation names, reverse navigations already listed by the parent (adding both sides makes JSON serialization loop), enum-typed properties, validation attributes, hand-picked sort columns, screens with detail grids or dependent drop-downs, and registration lines (DbSet, routes, sidebar).
