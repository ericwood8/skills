---
name: mysql
description: Use when connecting to, querying, or troubleshooting a MySQL database, running ad hoc SQL against one via CLI, or writing MySQL migration, seed or generator code. Trigger words - error 1045, error 1049, error 1146, error 1419, table name case on Windows vs Linux, DELIMITER, ROW_COUNT, enum column, CHECK_CLAUSE, Pomelo, trigger on a test container.
---

# MySQL

## Safety rules (apply whenever running SQL against MySQL)

1. **Idempotent where possible.** Prefer `INSERT IGNORE` / `ON DUPLICATE KEY UPDATE` over plain `INSERT`. Guard DDL with `CREATE TABLE IF NOT EXISTS`; MySQL has no `CREATE INDEX IF NOT EXISTS`, so declare indexes inline inside that block. For a table with no single-column natural key guard the seed `INSERT` with `WHERE NOT EXISTS (...)`. Guard natural-key seeds with `INSERT IGNORE` against a `UNIQUE` constraint.
2. **Single statement only.** Never send several statements separated by `;` in one execution (`SELECT 1; DROP TABLE Foo;`).
3. **Timeouts.** 10 s connection timeout (`--connect-timeout=10`). 30 s query timeout with `SET SESSION MAX_EXECUTION_TIME=30000;` or the hint `/*+ MAX_EXECUTION_TIME(30000) */`; it throttles only SELECT, not DML/DDL.
4. **Row cap.** Append `LIMIT 10000` to ad hoc SELECTs.
5. **Credentials.** Never echo a full connection string or password; redact (`Password=REDACTED`). `-p'password'` inline triggers the CLI warning "Using a password on the command line interface can be insecure"; the value still must not be echoed.

**Tool:** `python C:/Github/tools/sql_run.py --engine mysql --host <host> --user <user> --db <db> -e "<one statement>"` enforces rules 2 to 5 (refuses a second statement, 10 s / 30 s timeouts, 10000 row stop, password only via `MYSQL_PWD`, redacted output) and notes a non-idempotent statement. `--via <container>` borrows a running container's `mysql` client for any host. Read its docstring for the flags (`--read-only`, `--allow-multi` for a reviewed seed file, `--dry-run`).

## Connecting

- **No Windows Integrated Auth equivalent:** plan username/password auth in every environment. Use alphanumeric-only passwords for anything passed through shell commands (special characters broke CLI connections through shell escaping).
- **Borrow a running container's client to reach any target:** `docker exec <container> mysql -h <any host, including a remote RDS/Lightsail endpoint> -u <user> -p'<password>' <db> -e "<sql>"`. No native client needed on the host.
- **Credentials without a command line:** `MYSQL_PWD=... mysql -h 127.0.0.1 -u user db < file.sql` (the `mysql` client reads it; the .NET connectors do not: add it to the connection string from the environment at startup).

| Wrong | Right | Why |
|---|---|---|
| Reading **error 1045 "Access denied"** as a missing privilege and running `GRANT` | Check `SELECT user, host, plugin FROM mysql.user` and `SHOW GRANTS FOR ...` as root; `CREATE USER 'u'@'localhost' IDENTIFIED BY '...'; GRANT ALL ON db.* TO 'u'@'localhost';` | The user/host/password pair is not accepted (a missing account, a wrong password, another host, or the wrong user such as app-scoped vs master). `GRANT` on a user that does not exist fails with error 1410 ("You are not allowed to create a user with GRANT") in MySQL 8.4. A login with a grant on one database sees that database plus `information_schema` and `performance_schema` in `SHOW DATABASES`, and cannot `CREATE DATABASE` (error 1044) |
| Reading **error 1049 "Unknown database"** as a missing database | Use the schema name inside the instance | The cloud resource/instance name was used where the schema name is needed (two different names) |

## Case, names and escapes

| Wrong | Right | Why |
|---|---|---|
| `CREATE TABLE CustomerItem` on Windows and expecting the name back as PascalCase | Use `snake_case` and convert to Pascal in the generator. Check `SELECT @@lower_case_table_names`. `TABLE_SCHEMA` is the database name (MySQL has no separate schemas) | Windows stores table names lower case (`lower_case_table_names=1`): it is reported as `customeritem`, so a PascalCase name cannot be recovered. Column names keep their case |
| Code written against a Windows-created database with the wrong case, then run on Linux | One casing consistently: lowercase `snake_case` is the safe portable choice, or exact PascalCase if every query and mapping matches the DDL. Test on a throwaway container: `docker run -d --rm --name t -e MYSQL_ROOT_PASSWORD=x mysql:8.4`, then `docker stop t` | Linux is case-sensitive (`lower_case_table_names=0`; macOS `2`): after `CREATE TABLE Movies`, `FROM movies` fails with `ERROR 1146 Table 'db.movies' doesn't exist`, `USE CASETEST` on a database created as `casetest` fails with `ERROR 1049`, and `movies` and `Movies` can coexist as two tables. Column names, aliases and index names stay case-insensitive. The setting can only be chosen when the data directory is first initialized (8.0+ refuses to start if it is changed) |
| A generator writing string literals with only the quotes doubled (`'a\\b'`) | Double backslashes as well as quotes | A backslash in a string literal is an escape |
| The `utf8` character set | **`utf8mb4`** with `utf8mb4_unicode_ci` | `utf8` is a legacy 3-byte encoding |

- `DESC` in an index is honored as true descending on InnoDB since 8.0. There is no `INCLUDE` syntax: a plain composite index is the closest (InnoDB secondary indexes carry the primary key).

## Stored procedures and routines

- `CALL p(...)`, `IN`/`OUT`; a plain `SELECT` returns a result set. Errors with `SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = '...', MYSQL_ERRNO = 55509`; a foreign-key delete failure is error 1451 (`DECLARE EXIT HANDLER FOR 1451 SELECT -1`). Parameters shadow columns of the same name: prefix them.
- `CALL` returning a flag: `CASE WHEN EXISTS (...) THEN 1 ELSE 0 END AS IsSelected` arrives in the Oracle EF provider's `SqlQueryRaw<T>` as a `bool` property without a cast.

| Wrong | Right | Why |
|---|---|---|
| Default parameter values, `CREATE OR REPLACE PROCEDURE`, `RETURNING`, or `LIMIT` with an expression | Pass every parameter, in order, positionally (an optional create-user parameter is an explicit `NULL`, where SQL Server and PostgreSQL leave it out); `DROP PROCEDURE IF EXISTS`; `SELECT LAST_INSERT_ID()` or `SET v = LAST_INSERT_ID()`; `SET v_offset = (page-1)*size` first | MySQL has none of these; `LIMIT` takes parameters/variables but not expressions. A script also needs `DELIMITER $$ ... $$ DELIMITER ;` (a client feature, not SQL) |
| Testing "did the update find the row" with `ROW_COUNT()` | `EXISTS (SELECT 1 ...)` | `ROW_COUNT()` after an UPDATE is the number of rows that changed (0 for identical values) |
| `SET status='failed', retries=retries+1, run_after=IF(retries<max, ...)` | Put the column the others depend on last, or compute from the old value first | A `SET` list is evaluated left to right and later assignments see the new values (SQL Server and PostgreSQL read the old row), so `retries` is already incremented |

## Types and information_schema

- `tinyint(1)` is a boolean, a signed `tinyint` is -128..127, `unsigned` changes the C# type, `text`/`longtext`/`json` are unbounded, `datetime` has no zone, `timestamp` is converted to UTC; no `money` type (use `decimal(19,4)`).
- `information_schema.COLUMNS.COLUMN_TYPE` carries the length, unsigned and the enum list; `EXTRA` has `auto_increment`, `DEFAULT_GENERATED` (default is an expression), `STORED GENERATED`; `COLUMN_KEY = 'PRI'`; unique indexes from `STATISTICS` (`NON_UNIQUE = 0`); foreign keys from `KEY_COLUMN_USAGE` (`REFERENCED_TABLE_NAME IS NOT NULL`).

| Wrong | Right | Why |
|---|---|---|
| Reading an `enum('a','b')` list from `DATA_TYPE`, or splitting `COLUMN_TYPE` with `Split(',')` | Parse `COLUMN_TYPE` character by character. `CHARACTER_MAXIMUM_LENGTH` is the longest value | `DATA_TYPE` just says `enum`. A quote inside a value is **doubled** (`enum('it''s','a,b')`) and values may contain commas |
| A free-text input for an enum column | A drop-down. A `set` holds several values and does not fit a drop-down | A value outside the list is an error in strict mode (the default; the API returned 500). A procedure parameter typed `varchar(n)` writes into an enum column without a cast |
| Parsing `CHECK_CLAUSE` as written, or copying it into a create script | Normalise `\'` to `'` and drop `_charset` before parsing or running it (a `\'` outside a string is a syntax error, and a `_cp850` introducer names a character set the target server may not have). Use `information_schema.CHECK_CONSTRAINTS` (8.0.16 and later; an older server errors, catch it) joined to `TABLE_CONSTRAINTS` on `CONSTRAINT_SCHEMA`/`CONSTRAINT_NAME` | MySQL rewrites the clause with backticked names, lower-case `between`/`in`, and a character-set introducer with escaped quotes on every literal (the introducer follows the connection's character set: `_utf8mb4` or `_latin1`) (`` (`s` in (_utf8mb4\'Open\',_utf8mb4\'Closed\')) ``) |

## EF Core

- **Default to `Microting.EntityFrameworkCore.MySql` on the `MySqlConnector` driver.** It is Pomelo's fork under a new package and namespace (same API) and follows EF Core. Pomelo lagged a major EF Core release, and Oracle's `MySql.EntityFrameworkCore` (`UseMySQL(...)`) brings a second driver (`MySql.Data`, with its own `MySqlParameter`, `MySqlException` and `MySqlConnectionStringBuilder`), so the exception, parameter and builder types differ from the schema reader's `MySqlConnector`. Pomelo 9.0.0 (EF Core 9) cannot even be configured on EF Core 10.0.13 (`MissingMethodException`); Microting 10.0.14 targets EF Core 10 and needs `MySqlConnector` 2.6.2 or later.
- **The call:** `options.UseMySql(connectionString, serverVersion)`. `ServerVersion.AutoDetect(conn)` opens a connection while the options are built, so an app that must start with the server down uses a fixed `new MySqlServerVersion(new Version(8, 0, 0))`. The password from `MYSQL_PWD` is added with `MySqlConnector.MySqlConnectionStringBuilder` (neither driver reads that variable). The error number is `MySqlConnector.MySqlException.Number` (1062 duplicate, 1451 referenced, 1452 no parent, 1406 too long, 1048 null); the parameter type is `MySqlConnector.MySqlParameter`.
- **`BIT(64)` is a `ulong` to MySqlConnector.** An entity property typed `decimal` (the only type that holds 64 bits) fails on read with `Unable to cast object of type 'System.UInt64' to type 'System.Decimal'`; give it a converter, `new ValueConverter<decimal, ulong>(v => (ulong)v, v => v)`. `BIT(1)` reads as `bool` and `BIT(8)` as a number without help. A `SET` is its comma-separated text, `YEAR` a number, and spatial columns need a converter that turns well-known text into the binary form (a 4-byte SRID and the geometry); with `SRID 4326` the stored order is latitude first, so write the text with `ST_GeomFromText(..., 4326, 'axis-order=long-lat')` when the application means longitude first.
- `FromSqlRaw("CALL p(@a, @b)", params)` + `ToListAsync()` works; a `CALL` is not composable (no `.Count()`/`.Single()` in SQL). A scalar count procedure returns `SELECT COUNT(*) AS Value` for `SqlQueryRaw<int>`.

## Triggers and defaults

| Wrong | Right | Why |
|---|---|---|
| `CREATE TRIGGER` by a login limited to one database, claimed to work | Run the script as root or set `log_bin_trust_function_creators`. Say so in the script header and do not claim a trigger script works until it has run on a server | **Error 1419** ("You do not have the SUPER privilege and binary logging is enabled") for a login without `SUPER` while `log_bin_trust_function_creators` is 0. A login limited to one database cannot fix it |
| A test container trigger script run by a non-SUPER login (the Testcontainers user) | Start the container with the server option `skip-log-bin` (`WithCommand`); run trigger bodies between `DELIMITER` lines, which the `mysql` client reads | The `mysql:8.4` image logs in binary by default, so error 1419. Testcontainers.MySql's default user is `mysql@%` with no `SUPER` (`CREATE TRIGGER` fails with 1419), `WithCommand("--skip-log-bin")` fixes it, and `root` can create the trigger either way |
| `AFTER UPDATE OR DELETE` | One trigger per event (`T_U`, `T_D`), copying `OLD.<col>` | MySQL has no combined event |
| Running `information_schema.VIEWS.VIEW_DEFINITION` in another database as is | Strip the database name first | It qualifies tables with the database name |

## Disposable copies and throwaway servers

- **Copy a database by tables:** `CREATE DATABASE scratch;` then, per table, `CREATE TABLE scratch.t LIKE src.t; INSERT INTO scratch.t SELECT * FROM src.t;` (rows and secondary indexes came across in the test). A login that has a grant on one database only cannot run `CREATE DATABASE` (error 1044).
- **Test a routine when you cannot create a database:** create a copy of it under another name in the same database (`ZZ_<name>`; the script needs `DELIMITER $$`), call it between `START TRANSACTION;` and `ROLLBACK;`, then `DROP PROCEDURE` and check `information_schema.routines` for leftovers. Put each call that may be refused in its own file: the `mysql` client stops at the first error, so a refused call ends the script before the lines after it (tested: error 1644 at the `SIGNAL`, the later `SELECT` did not run). A good call inside the transaction raised the row count from 2 to 3 and `ROLLBACK` put it back to 2.
- **A throwaway MySQL server:** `docker run -d --rm --name <name> -e MYSQL_ROOT_PASSWORD=<throwaway> -p 127.0.0.1:3307:3306 mysql:8.4` on a spare port (never the user's containers); wait with `mysqladmin ping`; load the schema with `docker exec -i <name> mysql -uroot -p... < schema.sql`; remove it with `docker rm -f <name>`. Checked with 8.4.13. The saved tool `C:\Github\tools\throwaway_db.py` does all of this (`up`, `sql`, `list`, `down`, `--skip-log-bin`): it touches only `zz-` containers it made and never pulls an image, so prefer it to typing the commands.

## How this was verified

Tested on this machine. The evidence, the versions used and the items **not re-run** are in [reference/verification.md](reference/verification.md): read it before saying a claim here was tested, and treat anything listed there as not re-run as unverified.

## What this does not prove

- That a rule holds on MySQL 5.7, MariaDB, macOS (`lower_case_table_names=2`) or a managed service such as RDS. Tested versions: 8.0.46 on Windows and 8.4.13 on Linux.
- That a trigger script works: say so in the script header until it has run on the target server.
- That a generator's output is right for enum, `set` or `CHECK` forms not seen in the samples.
