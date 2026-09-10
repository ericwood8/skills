---
name: mysql
description: Safety guardrails for running ad hoc SQL against a MySQL database, plus concrete gotchas learned running one for the Critical Viewer project (Docker-borrowed CLI clients, real connection errors and their actual causes, an idempotent seed-script pattern, and MySQL/InnoDB quirks). Use when connecting to, querying, or troubleshooting a MySQL database, running ad hoc SQL against one via CLI, or writing MySQL-targeting migration/seed scripts.
---

# MySQL

## Safety rules — apply whenever running SQL against MySQL, not just this project

1. **Idempotent where possible.** Prefer `INSERT IGNORE` / `ON DUPLICATE KEY UPDATE` over plain `INSERT` for anything that might get re-run. Guard DDL with `CREATE TABLE IF NOT EXISTS` — MySQL has no `CREATE INDEX IF NOT EXISTS`, so declare indexes inline inside the `CREATE TABLE IF NOT EXISTS` block; they're covered by the same guard. For a table with no single-column natural key, guard the seed `INSERT` with an explicit `WHERE NOT EXISTS (...)` check instead.
2. **Single statement only.** Never send multiple statements separated by `;` in one execution — protects against an accidental or injected stacked statement (`SELECT 1; DROP TABLE Foo;`).
3. **Timeouts.** 10s connection timeout via `--connect-timeout=10` (or the connection string's equivalent). 30s query timeout via `SET SESSION MAX_EXECUTION_TIME=30000;` before running a query, or the inline hint `/*+ MAX_EXECUTION_TIME(30000) */` on a SELECT. Note `MAX_EXECUTION_TIME` only throttles SELECT statements — it does not bound DML/DDL execution time.
4. **Row cap.** Append `LIMIT 10000` to ad hoc SELECT queries run for inspection/debugging, so a huge result set doesn't end up dumped into context.
5. **Credential protection.** Never echo a full connection string or password anywhere the output might get displayed or logged — redact (`Password=REDACTED`) before showing any command or error output. The MySQL CLI itself already flags this risk: `-p'password'` inline triggers `mysql: [Warning] Using a password on the command line interface can be insecure` — expected for a one-off scripted command, but the value itself still shouldn't be echoed.

## Lessons learned (Critical Viewer project)

- **EF Core provider**: `Microting.EntityFrameworkCore.MySql`, a community Pomelo fork, was needed because upstream Pomelo hadn't cut an EF Core 10-compatible release yet. Check whether upstream has caught up before defaulting to the fork on a new project — API surface is meant to be identical, so swapping back should be low-risk.
- **No Windows Integrated Auth equivalent** for MySQL (unlike SQL Server) — always plan for username/password auth, in every environment including local dev.
- **Borrow a running container's `mysql` CLI client to reach any target**, not just that container's own database: `docker exec <local-container> mysql -h <any-host, including a remote AWS RDS/Lightsail endpoint> -u <user> -p'<password>' <db> -e "<sql>"`. Avoids installing a native MySQL client on the host machine at all — used successfully against both a local dev DB and a remote Lightsail-managed instance this way.
- **Real errors hit and their actual causes:**
  - `ERROR 1049 (42000): Unknown database 'X'` — came from confusing the *AWS resource/instance name* (e.g. `CriticalViewerDb-1`, what Lightsail calls the database instance) with the *actual schema name inside it* (e.g. `CriticalViewer`, what `USE`/connection strings need). Two different names, easy to conflate.
  - `ERROR 1045 (28000): Access denied for user 'X'@'Y' (using password: YES)` — was simply the wrong DB user's credentials (app-scoped user vs. master user), not a permissions bug. Double-check *which* user is intended before assuming misconfiguration.
- **Special characters in a password broke CLI connections** (shell-escaping issues) — removing them from the Lightsail-managed MySQL password fixed it immediately. Worth defaulting to alphanumeric-only passwords for anything that gets typed/passed via shell commands routinely.
- **Idempotent seed-script pattern that worked in practice**, safe to re-run any number of times: guard every `CREATE TABLE` with `IF NOT EXISTS` (inline indexes covered by the same guard), guard natural-key seed `INSERT`s with `INSERT IGNORE` against a `UNIQUE` constraint, and guard seed `INSERT`s for tables with no single-column natural key with an explicit `NOT EXISTS` subquery instead.
- `DESC` is honored as true descending order on InnoDB indexes since MySQL 8.0 — not just accepted-but-ignored as in older versions. Relevant when relying on index order instead of an explicit `ORDER BY ... DESC` at runtime.
- No equivalent to SQL Server's covering-index `INCLUDE` syntax — a plain composite index is the closest match, since InnoDB secondary indexes implicitly carry the primary key anyway.
- Use **`utf8mb4`** with `utf8mb4_unicode_ci`, not plain `utf8` — MySQL's `utf8` is a legacy 3-byte-max encoding, not actually full UTF-8.
