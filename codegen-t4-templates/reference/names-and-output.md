# Multi-file output and generated names

Read this when: a template writes several files, or a route, JSON name, navigation property or SQL name comes out wrong.

## Multi-file output and generated names

- Several files (an Angular component's css/html/spec/ts) or subfolders: emit `@@@FILE Models/Customer.cs@@@` before each file (`GeneratedFiles` splits; relative paths, no `..`, no drive; a marker followed directly by another gives an empty file). Point `-o` at the project root (Angular: `src\app`). Write only when text differs after normalising `\r\n`.
- API route: strip `E_` then `SY_`, lower-case, add `s` (`E_DonateLeave` is `/donateleaves`); the Angular dev proxy strips `/api`. The generator and every hand-written base must use the same `Pluralizer`.
- JSON names follow ASP.NET camel-casing (`FixCasing`): lower-case the first char and keep lower-casing while the next char is upper, stopping at the first char followed by a non-upper (`IsActive` is `isActive`, `SY_IsoCountry_Alpha3Code` is `sY_IsoCountry_Alpha3Code`, `E_TimeSheetId` is `e_TimeSheetId`). A TypeScript interface with the wrong case reads `undefined` silently.
- **A fixed route beside a table's `/{id}` route** (`export.csv`, `guards`, `lookup`, `junction/...`): FastAPI takes the first route that matches the path, so a fixed route must be declared before `/{id}` (a router listed first in `routers`, or earlier in the file); axum and ASP.NET prefer the fixed segment themselves.
- Navigation property: the FK column without its trailing `Id` (`DonateFrom_EmployeeId` becomes `DonateFrom_Employee`); skip it if it collides with a column.
- A naming rule that changes names changes the JSON the shared front ends read (Pascal-casing `require_customer_po` gives `RequireCustomerPo`, SQL Server has `RequireCustomerPO`): keep an `Acronyms=PO,UPC,MSRP` project list taken from the other database's capital runs.
- `NamingStyle=Pascal` converts at the schema reader; SQL keeps the real names. **A C#/TypeScript identifier uses `Name`; anything inside SQL text or `[Table]`/`[Column]` uses `DbName`** (`ColumnModel.DbName`, `TableModel.DbTableName`, `ForeignKeyModel.ReferencedDbTable/Columns/DisplayDbColumns`). EF allows one `[Column]` per property: one attribute carries real name and type. A PostgreSQL branch that wrote `Model.TableName` into SQL breaks for `snake_case` read with `NamingStyle=Pascal`.
