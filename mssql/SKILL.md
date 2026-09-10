---
name: mssql
description: Safety guardrails for running ad hoc T-SQL against a Microsoft SQL Server database, plus concrete gotchas learned running one for the Critical Viewer project (Windows Auth vs. containers, an efficient row-count trick via sys.partitions, an EF Core provider-swap testing gotcha, and a duplicate-row race pattern). Use when connecting to, querying, or troubleshooting a SQL Server database, running ad hoc T-SQL against one, or writing SQL Server-targeting migration/seed scripts.
---

# MS SQL Server

## Safety rules — apply whenever running SQL against SQL Server, not just this project

1. **Idempotent where possible.** SQL Server has no `CREATE TABLE IF NOT EXISTS` shorthand — guard DDL with `IF OBJECT_ID('dbo.TableName', 'U') IS NULL` before `CREATE TABLE`. For idempotent seed data, use `MERGE` or an explicit `IF NOT EXISTS (SELECT 1 FROM ... WHERE ...) INSERT ...` guard instead of a plain `INSERT`.
2. **Single statement only.** Never send multiple statements separated by `;` (or on separate lines with no meaningful `GO` batch separator between them) in one execution — protects against an accidental or injected stacked statement.
3. **Timeouts.** 10s connection timeout via the connection string's `Connection Timeout=10` (ADO.NET/EF Core) or equivalent. 30s query timeout via the client-side command timeout (`SqlCommand.CommandTimeout = 30` / EF Core's configured command timeout) — SQL Server has no direct server-side per-statement execution-time cap the way MySQL's `MAX_EXECUTION_TIME` hint works, so this has to be enforced client-side. For lock waits specifically, `SET LOCK_TIMEOUT 30000;` bounds how long a statement waits on a blocked resource before erroring out.
4. **Row cap.** Use `SELECT TOP 10000 ...` (or `OFFSET 0 ROWS FETCH NEXT 10000 ROWS ONLY` if the query already has an `ORDER BY`) on ad hoc SELECT queries run for inspection/debugging.
5. **Credential protection.** Never echo a full connection string or password anywhere the output might get displayed or logged — redact before showing any command or error output.

## Lessons learned (Critical Viewer project)

- **Windows Integrated Auth is a non-starter the moment a SQL Server workload might ever run in a container or the cloud.** This project started on SQL Server with Windows Auth against a specific domain-joined server, then had to migrate fully to MySQL once the deployment target became AWS Lightsail. If there's any chance of containers/cloud down the line, design for SQL Server (username/password) auth from the start rather than assuming Windows Auth and migrating later.
- The `mcr.microsoft.com/mssql/server` image is **Linux-based** (the container's own OS reports as Ubuntu) despite "SQL Server" reading as Windows-only in most people's mental model — it's run on Linux containers for years.
- To push a locally-tagged image to a registry expecting a specific name, `docker tag mcr.microsoft.com/mssql/server:<version> <local-name>:local` is a plain rename/tag operation — no rebuild needed.
- **Efficient row-count without a table scan**: query `sys.partitions` instead of `COUNT(*)` — reads from SQL Server's internal metadata rather than scanning the table. Wired up via EF Core's `SqlQueryRaw<int?>`, aliasing the result column as `Value` so EF Core's scalar-mapping convention picks it up.
- **EF Core provider-swap gotcha in tests**: `AddDbContext` registers its configuration as an *additive* `IDbContextOptionsConfiguration<T>` service. Removing the `DbContextOptions<TContext>` registration alone is **not** enough to swap SQL Server for the InMemory provider in xUnit tests — the old SQL Server configuration is still separately registered as `IDbContextOptionsConfiguration<TContext>` and collides with the new one. Both registrations have to be removed for the swap to actually take effect.
- **Duplicate-row race under concurrent requests** (enforcing "one row per natural key" beyond what a UNIQUE constraint alone conveniently reports): pre-check for the duplicate *and* wrap the insert in `catch (DbUpdateException)` as a fallback, re-querying inside the catch block to confirm it really was a duplicate before treating it as one. A catch *filter* can't `await`, so the confirming re-query has to happen in the catch body, rethrowing if it turns out not to have been a duplicate after all.
- Used `DATETIME2(0)` (zero fractional-second precision) rather than `DATETIMEOFFSET` for timestamps — a deliberate choice when the application already normalizes everything to UTC itself and doesn't need the database to carry timezone-offset info per row.
