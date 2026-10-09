---
name: mysql
description: Safety rules for running ad hoc SQL against MySQL, plus concrete gotchas: Docker-borrowed CLI clients, connection errors and their real causes, idempotent seed scripts, credentials, table-name case on Windows vs Linux, escapes, stored procedure limits, types and information_schema, EF Core providers, enum and CHECK columns, triggers and defaults. Use when connecting to, querying, or troubleshooting a MySQL database, running ad hoc SQL against one via CLI, or writing MySQL migration/seed scripts or generators.
---

# MySQL

## Safety rules (apply whenever running SQL against MySQL)

1. **Idempotent where possible.** Prefer `INSERT IGNORE` / `ON DUPLICATE KEY UPDATE` over plain `INSERT`. Guard DDL with `CREATE TABLE IF NOT EXISTS`; MySQL has no `CREATE INDEX IF NOT EXISTS`, so declare indexes inline inside that block. For a table with no single-column natural key guard the seed `INSERT` with `WHERE NOT EXISTS (...)`. Guard natural-key seeds with `INSERT IGNORE` against a `UNIQUE` constraint.
2. **Single statement only.** Never send several statements separated by `;` in one execution (`SELECT 1; DROP TABLE Foo;`).
3. **Timeouts.** 10 s connection timeout (`--connect-timeout=10`). 30 s query timeout with `SET SESSION MAX_EXECUTION_TIME=30000;` or the hint `/*+ MAX_EXECUTION_TIME(30000) */`; it throttles only SELECT, not DML/DDL.
4. **Row cap.** Append `LIMIT 10000` to ad hoc SELECTs.
5. **Credentials.** Never echo a full connection string or password; redact (`Password=REDACTED`). `-p'password'` inline triggers the CLI warning "Using a password on the command line interface can be insecure"; the value still must not be echoed.

## Connecting

- **No Windows Integrated Auth equivalent:** plan username/password auth in every environment. Use alphanumeric-only passwords for anything passed through shell commands (special characters broke CLI connections through shell escaping).
- **Borrow a running container's client to reach any target:** `docker exec <container> mysql -h <any host, including a remote RDS/Lightsail endpoint> -u <user> -p'<password>' <db> -e "<sql>"`. No native client needed on the host.
- **Credentials without a command line:** `MYSQL_PWD=... mysql -h 127.0.0.1 -u user db < file.sql` (the `mysql` client reads it; the .NET connectors do not: add it to the connection string from the environment at startup).
- **Error 1045 "Access denied"** means the user/host/password pair is not accepted (a missing account, a wrong password, another host, or the wrong user such as app-scoped vs master), not a missing privilege: `GRANT` on a user that does not exist does nothing in MySQL 8. Check `SELECT user, host, plugin FROM mysql.user` and `SHOW GRANTS FOR ...` as root; `CREATE USER 'u'@'localhost' IDENTIFIED BY '...'; GRANT ALL ON db.* TO 'u'@'localhost';`. A grant on one database shows only that database in `SHOW DATABASES`, and such a login cannot `CREATE DATABASE`.
- **Error 1049 "Unknown database"** is often the cloud resource/instance name used where the schema name inside it is needed (two different names).

## Case, names and escapes

- **Windows stores table names lower case** (`lower_case_table_names=1`): `CREATE TABLE CustomerItem` is reported as `customeritem`, so a PascalCase name cannot be recovered; use `snake_case` and convert to Pascal in the generator. Column names keep their case. Check `SELECT @@lower_case_table_names`. `TABLE_SCHEMA` is the database name (MySQL has no separate schemas).
- **Linux is case-sensitive** (`lower_case_table_names=0`; macOS `2`): after `CREATE TABLE Movies`, `FROM movies` fails with `ERROR 1146 Table 'db.movies' doesn't exist`, `USE CASETEST` on a database created as `casetest` fails with `ERROR 1049`, and `movies` and `Movies` can coexist as two tables. Column names, aliases and index names stay case-insensitive. The setting can only be chosen when the data directory is first initialized (8.0+ refuses to start if it is changed). Code written against a Windows-created database with the wrong case works on Windows and breaks on Linux: use one casing consistently (lowercase `snake_case` is the safe portable choice, or exact PascalCase if every query and mapping matches the DDL). Test on a throwaway container: `docker run -d --rm --name t -e MYSQL_ROOT_PASSWORD=x mysql:8.4`, then `docker stop t`.
- **A backslash in a string literal is an escape** (`'a\\b'`): a generator writing literals must double backslashes as well as quotes.
- Use **`utf8mb4`** with `utf8mb4_unicode_ci`, not `utf8` (a legacy 3-byte encoding).
- `DESC` in an index is honored as true descending on InnoDB since 8.0. There is no `INCLUDE` syntax: a plain composite index is the closest (InnoDB secondary indexes carry the primary key).

## Stored procedures and routines

- `CALL p(...)`, `IN`/`OUT`; a plain `SELECT` returns a result set. No default parameter values (pass every parameter, in order, positionally; an optional create-user parameter is an explicit `NULL`, where SQL Server and PostgreSQL leave it out). No `CREATE OR REPLACE PROCEDURE` (`DROP PROCEDURE IF EXISTS`). A script needs `DELIMITER $$ ... $$ DELIMITER ;` (a client feature, not SQL). No `RETURNING` (`SELECT LAST_INSERT_ID()` or `SET v = LAST_INSERT_ID()`). `LIMIT` takes parameters/variables but not expressions (`SET v_offset = (page-1)*size` first). Errors with `SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = '...', MYSQL_ERRNO = 55509`; a foreign-key delete failure is error 1451 (`DECLARE EXIT HANDLER FOR 1451 SELECT -1`). Parameters shadow columns of the same name: prefix them.
- **`ROW_COUNT()` after an UPDATE is the number of rows that actually changed** (0 for identical values): test existence with `EXISTS (SELECT 1 ...)`.
- `CALL` returning a flag: `CASE WHEN EXISTS (...) THEN 1 ELSE 0 END AS IsSelected` arrives in the Oracle EF provider's `SqlQueryRaw<T>` as a `bool` property without a cast.

## Types and information_schema

- `tinyint(1)` is a boolean, a signed `tinyint` is -128..127, `unsigned` changes the C# type, `text`/`longtext`/`json` are unbounded, `datetime` has no zone, `timestamp` is converted to UTC; no `money` type (use `decimal(19,4)`).
- `information_schema.COLUMNS.COLUMN_TYPE` carries the length, unsigned and the enum list; `EXTRA` has `auto_increment`, `DEFAULT_GENERATED` (default is an expression), `STORED GENERATED`; `COLUMN_KEY = 'PRI'`; unique indexes from `STATISTICS` (`NON_UNIQUE = 0`); foreign keys from `KEY_COLUMN_USAGE` (`REFERENCED_TABLE_NAME IS NOT NULL`).
- **`enum('a','b')`:** `COLUMN_TYPE` is the only place the list is (`DATA_TYPE` just says `enum`). A quote inside a value is **doubled** (`enum('it''s','a,b')`) and values may contain commas: parse character by character, not with `Split(',')`. `CHARACTER_MAXIMUM_LENGTH` is the longest value. A value outside the list is an error in strict mode (the default; the API returned 500), so a form should offer a drop-down. A `set` holds several values and does not fit a drop-down. A procedure parameter typed `varchar(n)` writes into an enum column without a cast.
- **CHECK constraints:** `information_schema.CHECK_CONSTRAINTS` (8.0.16 and later; an older server errors, catch it) joined to `TABLE_CONSTRAINTS` on `CONSTRAINT_SCHEMA`/`CONSTRAINT_NAME` gives `CHECK_CLAUSE`: MySQL's own rewrite with backticked names, lower-case `between`/`in`, and a character-set introducer with escaped quotes on every literal (`` (`s` in (_utf8mb4\'Open\',_utf8mb4\'Closed\')) ``). Normalise `\'` to `'` and drop `_charset` before parsing.

## EF Core

- Pomelo.EntityFrameworkCore.MySql lagged a major EF Core release; the community fork `Microting.EntityFrameworkCore.MySql` or Oracle's `MySql.EntityFrameworkCore` (`UseMySQL(...)`, its own `MySql.Data.MySqlClient.MySqlParameter` and `MySqlConnectionStringBuilder`) follow EF Core. Check whether upstream has caught up before defaulting to a fork (the API surface is meant to be identical).
- `FromSqlRaw("CALL p(@a, @b)", params)` + `ToListAsync()` works; a `CALL` is not composable (no `.Count()`/`.Single()` in SQL). A scalar count procedure returns `SELECT COUNT(*) AS Value` for `SqlQueryRaw<int>`.

## Triggers and defaults

- `CREATE TRIGGER` fails with **error 1419** ("You do not have the SUPER privilege and binary logging is enabled") for a login without `SUPER` while `log_bin_trust_function_creators` is 0. A login limited to one database cannot fix it: run the script as root or set the variable. Say so in the script header and do not claim a trigger script works until it has run on a server.
- `DEFAULT (USER())` is refused (**error 1674**, not deterministic for the replication format): use a constant default (`'system'`) and set the real user in a trigger with `CURRENT_USER()`.
- MySQL has no `AFTER UPDATE OR DELETE`: one trigger per event (`T_U`, `T_D`), copying `OLD.<col>`.
