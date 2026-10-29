# SQL and dialect facts

Read this when: a template writes SQL or calls a routine, or one database behaves differently from the others.

## Database facts live in the database skills

This file keeps what the generator decides about SQL. For how a database itself behaves, read the skill for it:

- `mssql`: T-SQL, `sys.partitions` counts, `UPDATE ... SET` duplicates, triggers, IDENTITY rebuild, EF Core and SQL Server (`OUTPUT` with triggers, `EF.Constant`, row goal).
- `postgres`: quoted PascalCase names, `RETURNS SETOF` functions and the `AS "Value"` scalar, parameter order and defaults, function overloads, NULL `information_schema` booleans, `timestamptz`, enum casts, CHECK shapes.
- `mysql`: no default parameters, no `CREATE OR REPLACE PROCEDURE`, no `RETURNING`, backslash escapes, lower-case table names on Windows, `ROW_COUNT()`, enum and CHECK columns, the Pomelo and Oracle providers.
- `efcore-aspnet-review`: `SqlQueryRaw<T>` against `EXEC` is not composable, and a scalar is read from a column called `Value`.

## What the templates do

- **Sorting from user input without dynamic SQL:** compare the column name with a fixed list inside one `ORDER BY` of `CASE WHEN @SortColumn = N'Col' AND @SortDescending = 0 THEN col END ASC, CASE WHEN ... = 1 THEN col END DESC` per column and direction, then a fixed default order. Test with `x' OR 1=1 --`. A foreign key sorts by the parent's display column with a scalar subquery; alias the main table and qualify columns.
- **Adding parameters to a PostgreSQL function leaves the old one behind** ("function is not unique", see the `postgres` skill): generate `DROP FUNCTION IF EXISTS "s"."f"(<old types>)` first. A MySQL signature change means regenerating the callers.
- **Calling a clone routine from EF:** SQL Server `OUTPUT` parameter with **named** arguments; PostgreSQL returns the key as the function value (`SqlQueryRaw<int>("SELECT ... AS \"Value\"")`, `=>` named notation); MySQL ends the procedure with `SELECT v_new_key AS \`Value\`` and the caller passes every argument in order. `SqlQueryRaw<T>` reads a scalar from a column called `Value`.
- A copy must not trip a unique column: give each unique text column a free value first (`SuggestUnique<Column>`, cut the source short so the suffix fits); offer clone only when all unique columns are text and the database assigns the key.
- **Apply the SQL after a signature change** (`APPLY_SQL=1`) and query the catalog (`sys.parameters`, `pg_proc`, `information_schema.PARAMETERS`) to confirm, or generated callers fail at run time with "too many arguments" while everything builds. Generated search procedures list columns, so re-create them after a column change.
- PostgreSQL enum columns: EF sends `text`, so the generated SQL creates an assignment cast (`SP_EnumCasts` writes it; the cast itself is in the `postgres` skill). Cast to `::text` for `ILIKE`/ordering. A CHECK list is `= ANY ((ARRAY['A'::character varying, ...])::text[])`: parse only that shape; set `maxLength` to the longest value on an unbounded `text` column. A unique enum column cannot be cloned.
- A lookup table is not always an enum: a small `*Status` table is derived as an enum (no entity, repository, API); set `EnumTables=none` to show its description.
