---
name: odata-generated-grids
description: Use when adding a grid feature to the CodeGenNew page templates, touching OData query building, changing which columns a generated React, Angular or Blazor grid shows, or verifying a grid change in a running app. Trigger words - OData variant, $select, ChosenGridColumns, hidden column sort, owner-scoped table, REST search refused.
---

# OData and REST grids in CodeGenNew

## Shape of the templates
- The OData page is the **same template run with `Variant=OData`** (`TSX_Page`, `TSX_DetailMasterPage`, `TS_Component`, `TS_DetailMasterComponent`, `BLZ_Page`). A grid feature is one edit with an `if (odata)` branch, never a copy of the template. Update the template header comment and `Docs/TemplateNotes/<Name>_v1.md` for each of the five.
- Essentials (`PagedCrudScreen`, `ODataPagedScreen`) are written once and kept, so older projects hold the old file. Put new behavior in the page template or in a plan template (database-level, rewritten every run). Example: Angular `GridColumns` keeps `sortable` and `properties` as arrays updated in place and the screen hands them over once, so no essentials file changed.
- Shared support code (`components/gridColumns.tsx`, `grid-columns.ts`, `Services/GridColumns.cs`) is a `DatabaseOnly` template per stack; a rule all three must agree on lives in Core (`ChosenGridColumns`: Offered, Initial, Sortable, Resolve, Toggle, CutLength) with one test that reads the lists each stack wrote for the same table and compares them with the rule.

## Behavior rules to keep
- Hiding a column drops a sort on it and a filter typed for it. REST re-reads without the sort (`clearSort()` / `sortBy = null`); OData filters the sorts, resets the filters of the hidden field and re-reads.
- OData builds `$select` from the shown columns at **every** read. Edit reads the whole row by its key, so a hidden key or column never reaches the form.
- The last shown column cannot be cleared; a saved name the table no longer has is dropped; a stored value that is not a list is ignored; the key and credential columns are never offered.
- The choice is per browser and grid: `localStorage` `gridColumns:<gridKey>` (React, Angular), `<project>.columns.<pageclass>` (Blazor). Name/active lists get no picker.
- REST sorts by every column except long text and binary; REST grids have no filter row; the form already reads every parent list, so a foreign key shown late needs no extra call. Check such assumptions against the code before building from an item's text: the item 118 text was wrong on three of them.

## Owner scope and roles
- On a table scoped to its owner, OData applies the scope whatever `$select` asks for; the REST search is refused with `/problems/search-not-scoped`; a REST read answers whole rows for one's own record and refuses another person's.
- `Docs/Verification/Test-ODataScoping.js` (39 checks, three real sign-ins on a scratch copy) is the proof. Add a check there for any change that widens what a request can ask for.

## Proving a grid change
1. Unit suite (Core rule test, same-lists test, the template tests that assert the generated text).
2. Scratch copy of each sample with `OData=true` (React: `npx tsc -b` + `vitest`; Angular: `ng build` + `ng test`; Blazor: both projects build) and the API on a **scratch database**; see `verify-in-running-app`.
3. Click through on a wide table (Customer, 31 columns): tick and clear a column, hide the sorted column (next request has no sort), type a filter then hide its column (next request has none), read `$select` from the network request, reload (choice kept), store a bad value (ignored), press Edit (whole row).
4. Angular: `Docs/Verification/Test-AngularVersion.ps1 -Version 18` (Karma/Jasmine) as well as the current version; generated specs may use only `describe`, `it`, `expect`, `beforeEach`. `AdaptAngularHtml` turns `*ngIf` into `@if` for Angular 21+, so tests assert both forms.
5. Switch the fix off once (remove `clearSort()`), see the generated test fail, restore.

## Template traps seen here
- A `#>` inside a T4 string ends the block: build it from pieces.
- An import used only by generated tests (`fireEvent`) must be conditional on the same flag that writes the tests; helper definitions must be declared before first use.
- Blazor OData keeps the last Search's conditions while paging, so hiding a column re-reads the filter boxes and applies text typed but not yet searched; React and Angular apply typed text at every read. Report such differences instead of hiding them.
- Blazor does not render name/active lists; WinUI3 grids read the database directly and have no OData or Columns list.
