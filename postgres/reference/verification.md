# How the postgres skill was verified

Evidence behind the claims in [SKILL.md](../SKILL.md). Moved out of it so that it is not read on every load.

- Written while building CodeGenNew's PostgreSQL support and samples (first commit 2026-10-02, last change 2026-10-22).
- Re-checked on 2026-10-29 on PostgreSQL 18.6 with the `psql` 18 client, read-only or in `pg_temp`: `information_schema` boolean expressions come back NULL (182 columns of the InvoiceSystem copy); a function with a parameter lacking a default after one with a default is refused (`input parameters after one with a default value must also have defaults`); `localhost` is accepted while the machine name is refused by `pg_hba.conf`; `reltuples` is -1 for a table that was never analyzed.
- Also checked in `pg_temp`, rolled back: `CREATE OR REPLACE FUNCTION` with a new parameter list leaves two functions and an ambiguous call fails ("is not unique"); an enum column rejects a `text` value until the assignment cast exists, after which the insert works and `m::text ILIKE` matches.
- Not re-run: the Npgsql and EF Core items (timestamp, `jsonb`, `money`), psycopg and SQLAlchemy, CHECK parsing, trigger history, `setval`.
