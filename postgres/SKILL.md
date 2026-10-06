---
name: postgres
description: Safety guardrails for running ad hoc SQL against PostgreSQL, plus concrete gotchas learned building CodeGenNew's PostgreSQL support — reading the schema from information_schema and pg_catalog, functions instead of stored procedures, quoted PascalCase names, Npgsql timestamp/UTC and NULL handling, passwords via PGPASSWORD, psql from Git Bash. Use when running SQL against a PostgreSQL server, writing PostgreSQL DDL/functions, reading its catalog, or wiring EF Core/Npgsql to it.
---

# PostgreSQL

## Safety rules — apply whenever running SQL against PostgreSQL
1. **Never put a password in a file, script, doc, test or commit.** Pass it per command: `PGPASSWORD=... psql ...` (psql and Npgsql both read `PGPASSWORD`), or a connection string with no password plus the environment variable. A password the user gives in chat is for that session's commands only unless they say where it is recorded (their private global CLAUDE.md, never a repo).
2. **Test inside a transaction you roll back**: `BEGIN; ... ROLLBACK;` in the `-f` script. Remember that sequences (`GENERATED ... AS IDENTITY`, `serial`) are **not** rolled back, so ids keep climbing.
3. One statement per ad hoc `-c`; for anything longer use `-f file.sql` with `-v ON_ERROR_STOP=1 -q`. Without `ON_ERROR_STOP` a failed statement inside `BEGIN` aborts the transaction and every later line says "current transaction is aborted", hiding the first error: read the first error only.
4. Bound inspection queries (`LIMIT`); `SET statement_timeout = '30s'` for anything that might scan.
5. psql from Git Bash: `"/c/Program Files/PostgreSQL/18/bin/psql.exe" -h localhost -U <user> -d <db> -f file.sql`. `localhost` works while the machine name may be refused by `pg_hba.conf` (`no pg_hba.conf entry for host ...`): use the address the user has allowed.

## Names
- Unquoted names fold to **lower case**; `"CustomerId"` keeps its case and must then be quoted everywhere (`"public"."Customer"`). A PascalCase sample database works if every identifier is quoted; real databases are usually `snake_case`.
- A parameter or variable named like a column makes plpgsql say "column reference is ambiguous". Prefix parameters (`"pName"`), or write `#variable_conflict use_column` at the top of the body.

## Reading the schema (what a code generator needs)
- `information_schema.tables` / `columns` (portable): `udt_name` (`int4`, `bool`, `varchar`, `timestamp`, `timestamptz`, `numeric`, `uuid`, `bytea`, `jsonb`), `character_maximum_length`, `numeric_precision/scale`, `is_nullable`, `column_default`, `is_identity` / `identity_start`, `is_generated` / `generation_expression`.
- Keys, unique indexes and foreign keys come from `pg_catalog`: `pg_index` (`indisprimary`, `indisunique`, `indnkeyatts` = key columns without INCLUDE columns, `indkey[0]` is 0-based), `pg_constraint` (`contype = 'f'`; `conkey` / `confkey` are parallel arrays: `unnest(conkey, confkey) WITH ORDINALITY` pairs the columns of a composite key), `pg_class` + `pg_namespace`. `pg_indexes` only gives index definitions as text.
- A `serial` column has no identity flag: it is a `nextval('seq'::regclass)` default. Treat both as identity.
- **`information_schema` booleans can be NULL**: `(c.is_identity = 'YES' OR c.column_default LIKE 'nextval(%')` is NULL when the default is NULL, and a reader's `GetBoolean` throws "Column 'x' is null". Wrap in `COALESCE(..., false)`. Only a live test found it.
- Constraint names are unique **per table**, not per schema: group foreign keys by (constraint, other table).
- `reltuples` is -1 until a table is analyzed and stale after a bulk load: count for real (`SELECT count(*) FROM (SELECT 1 FROM t LIMIT 10001) x`) for small tables.

## Functions instead of stored procedures
- Search/list: `CREATE OR REPLACE FUNCTION ... RETURNS SETOF "public"."T" LANGUAGE sql STABLE` then `SELECT * FROM f(...)` (composable, EF maps it like the table). Scalar count: `RETURNS integer`, called `SELECT f(...) AS "Value"` so EF's `SqlQueryRaw<int>` finds the column.
- Parameters without a default must come **before** parameters with one; call optional ones by name (`"pName" => 'x'`). `OFFSET n LIMIT m` or `LIMIT m OFFSET n`; case-insensitive search is `ILIKE`.
- No OUTPUT parameters: return the new key (`RETURNING "Id" INTO v_id; RETURN v_id`). Errors: `RAISE EXCEPTION '...' USING ERRCODE = '55509'`. A foreign-key delete failure is the `foreign_key_violation` exception (`EXCEPTION WHEN foreign_key_violation THEN ...`; the block makes a sub-transaction). The function runs in the caller's transaction: there is no `BEGIN TRANSACTION` inside it.
- Upsert: `UPDATE ...; IF FOUND THEN ... END IF; INSERT ...` or `ON CONFLICT (key) DO NOTHING`; after loading rows with explicit identity values, move the counter: `SELECT setval(pg_get_serial_sequence('"public"."T"', 'Id'), (SELECT max("Id") FROM "public"."T"))`. `OVERRIDING SYSTEM VALUE` is needed to insert into a `GENERATED ALWAYS` identity.

## Npgsql / EF Core
- A `DateTime` maps to **`timestamp with time zone`** and Npgsql refuses any non-UTC value ("Cannot write DateTime with Kind=Unspecified to PostgreSQL type 'timestamp with time zone'"). For `timestamp` columns say so: `[Column(TypeName = "timestamp")]` or `configurationBuilder.Properties<DateTime>().HaveColumnType("timestamp")` in `ConfigureConventions`.
- `FromSqlRaw("SELECT * FROM f(@p1, @p2)", new NpgsqlParameter("@p1", NpgsqlDbType.Text) { Value = (object?)x ?? DBNull.Value })`: type a null parameter. The `money` type works as `decimal` (`[Column(TypeName = "money")]`) but real databases use `numeric`.
- Entities with `jsonb` columns need `[Column(TypeName = "jsonb")]` on the string property.

## CHECK constraints (schema reading, 2026-10)

`pg_get_constraintdef(oid)` gives `CHECK (((points >= 0) AND (points <= 10)))`; `BETWEEN` comes back expanded to two comparisons, a numeric literal can carry a cast (`(0)::numeric`) and an `IN` list comes back as `= ANY (ARRAY['a'::text, 'b'::text])`. Strip casts and parentheses before parsing. A strict bound (`> 0`) must stay strict: a whole-number column turns it into the next integer, a decimal box treats it as inclusive and the database still refuses the edge. `CREATE DATABASE x TEMPLATE y` fails while anybody is connected to `y`: terminate the sessions first (`pg_terminate_backend`).

## Trigger history (2026-10)

One function and one `AFTER UPDATE OR DELETE ... FOR EACH ROW` trigger: `TG_OP` says `UPDATE` or `DELETE`, `OLD.<col>` is the before-image, `current_user` is the login, `now() AT TIME ZONE 'utc'` the UTC time. Quote PascalCase names in the function body too.
