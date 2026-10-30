---
name: mssql
description: Use when connecting to, querying, or troubleshooting a SQL Server database, running ad hoc T-SQL, or writing SQL Server migration, seed or schema scripts. Trigger words - sqlcmd, bcp, Msg 264, row goal, slow OFFSET FETCH, IDENTITY rebuild, schema.sql, seed.sql, trigger OUTPUT clause.
---

# MS SQL Server

## Safety rules (apply whenever running SQL against SQL Server)

1. **Idempotent where possible.** No `CREATE TABLE IF NOT EXISTS`: guard DDL with `IF OBJECT_ID('dbo.TableName', 'U') IS NULL`. For seed data use `MERGE` or `IF NOT EXISTS (SELECT 1 FROM ... WHERE ...) INSERT ...`, not a plain `INSERT`.
2. **Single statement only.** Never send several statements separated by `;` (or on separate lines with no `GO`) in one execution: it protects against an accidental or injected stacked statement.
3. **Timeouts.** 10 s connection timeout (`Connection Timeout=10`). 30 s query timeout client-side (`SqlCommand.CommandTimeout = 30` / EF Core's command timeout): SQL Server has no server-side per-statement cap like MySQL's `MAX_EXECUTION_TIME`. `SET LOCK_TIMEOUT 30000;` bounds how long a statement waits on a blocked resource.
4. **Row cap.** Ad hoc inspection SELECTs use `SELECT TOP 10000 ...` (or `OFFSET 0 ROWS FETCH NEXT 10000 ROWS ONLY` with an `ORDER BY`).
5. **Credentials.** Never echo a full connection string or password; redact before showing any command or error output.

**Tool:** `python C:/Github/tools/sql_run.py --engine mssql --host <server> --trusted|--user <login> --db <db> -e "<one statement>"` enforces rules 2 to 5 (refuses a second statement and `GO`, 10 s / 30 s timeouts, 10000 row stop, password only via `SQLCMDPASSWORD`, redacted output) and notes a non-idempotent statement. Read its docstring for the flags (`--read-only`, `--via <container>`, `--dry-run`).

## Design and EF Core notes

- **Windows Integrated Auth does not work once the workload may run in a container or the cloud.** If that is possible, design for username/password auth from the start.
- The `mcr.microsoft.com/mssql/server` image is **Linux-based**. Retagging it for a registry push is in the `docker` skill.
- `DATETIME2(0)` instead of `DATETIMEOFFSET` is reasonable when the application normalizes everything to UTC itself.

| Wrong | Right | Why |
|---|---|---|
| `COUNT(*)` to get a row count | Query `sys.partitions`. Through EF Core use `SqlQueryRaw<int?>` and alias the column as `Value` | A table scan; the scalar mapping only picks up a column called `Value` |
| The same column twice in one `UPDATE ... SET`, from two independent `IF`s that can both fire | At most one assignment per column: `IF <force-condition> SET col = <forced> ELSE IF <caller supplied a value> SET col = <caller's value>` | `Msg 264`: "specified more than once in the SET clause". There is no "last write wins" |

## Exporting data and schema

| Wrong | Right | Why |
|---|---|---|
| `sqlcmd ... -E -U <login>` (both on the command line) | Pass `-E` or `-U`/`-P`, not both | `Sqlcmd: The -E and the -U/-P options are mutually exclusive.` Same message from the ODBC `sqlcmd` 16.0.1000.6 and go-sqlcmd 1.10.0. `SQLCMDUSER`/`SQLCMDPASSWORD` set in the environment did **not** trigger it with either: `-E` still used the Windows login |
| `sqlcmd -E -i C:/Users/me/x.sql` (a forward-slash absolute path) | A relative path (`-i x.sql`) or backslashes (`-i 'C:\Users\me\x.sql'`) | The ODBC `sqlcmd` 16.0.1000.6 reads the path as `C:` plus switches: `/Users` becomes `-U`, so you get the **same misleading message** "The -E and the -U/-P options are mutually exclusive"; `C:/Specs/x.sql` gave "Error occurred while opening or operating on file C: (Reason: Access is denied)". Relative and backslash paths worked from Git Bash and from PowerShell |
| `sqlcmd` to export long generated text | `bcp "SELECT ..." queryout file.txt -c -T -S server -d db -r '\n'` | `sqlcmd` cuts `varchar(max)` and other max-width columns at 256 characters by default (a plain `varchar(8000)` printed in full). `-y 0` removes the cut but is refused with `-h`; with `-h -1` use `-y 8000`. `bcp` has no width limit (an 8000-character value came out whole); rows with embedded newlines span lines (4 rows with an embedded LF and CRLF came out as 6 lines, and on Windows `-r '\n'` writes the row end as CRLF, bytes `0d0a`). To keep rows intact use terminators that cannot occur in the data, for example `-t '~\|~' -r '~#~'`; the file split cleanly on `~#~` and `bcp in` with the same terminators loaded all 4 rows back with their newlines |

- To copy a schema to another database, dump `sys.tables/columns/types/default_constraints`, `sys.foreign_keys(+_columns)` and `sys.indexes(+_columns)` with `sqlcmd -W -h -1 -s "|"` into small files and generate the other DDL from them; generate `INSERT`s with a `SELECT` that builds the literals (`REPLACE(x, '''', '''''')`, `CONVERT(varchar(23), d, 121)`, `CASE bit WHEN 1 ...`) and run it through `bcp`.
- **Output for generated text:** `-y 0` (alone, never with `-h`) prints no header line; trim trailing spaces per line and add `-f 65001` for UTF-8. Prove a schema/seed script pair by creating an empty database, running both twice, regenerating the schema from the result and diffing it against the original. A view or procedure created with `CREATE OR ALTER` is stored with the words `OR ALTER` blanked to spaces, so the regenerated text differs from the first only by spaces and a trailing blank line: normalise whitespace before the diff (tables, keys, indexes, defaults, checks and foreign keys came back identical).
- **Backup/restore for a scratch copy:** `BACKUP DATABASE X TO DISK = N'<path>' WITH COPY_ONLY, INIT` (copy-only so the real backup chain is untouched), then `RESTORE DATABASE X_scratch FROM DISK = ... WITH MOVE '<logical data name>' TO '<new .mdf>', MOVE '<logical log name>' TO '<new .ldf>'`. The service account writes the file, so use `SERVERPROPERTY('InstanceDefaultBackupPath')` (and `InstanceDefaultDataPath` for the new files), not a user temp folder, and get the logical names from `RESTORE FILELISTONLY`. `sqlcmd -b` plus `$LASTEXITCODE` is how a PowerShell script learns of a failure (a failing native command does not stop the script). Drop the copy with `ALTER DATABASE X_scratch SET SINGLE_USER WITH ROLLBACK IMMEDIATE; DROP DATABASE X_scratch;`, list `sys.databases` afterwards, and delete the `.bak` from the backup folder (a plain `rm` from the shell worked); never enable `xp_cmdshell` for this. Checked with `InvoiceSystem` (23 tables in the copy, as in the original).

## Scripts written from the catalog

- **CHECK constraints:** `sys.check_constraints` joined to `sys.columns` on `parent_object_id`/`parent_column_id`. A table-level check mentioning one column has `parent_column_id = 0`: find the column by reading the definition. `IN ('a','b')` is stored as an OR chain in the catalog's own order (`([s]='Closed' OR [s]='On hold' OR [s]='Open')`). Skip `is_disabled = 1`.
- **A key sequence table** (`GetNextID`) must lock the whole table (`WITH (TABLOCKX)`) inside its transaction, so every insert of every listed table queues behind one counter; say so next to the script. Seed each counter from `MAX(key)`.
- **Triggers that call a linked server run inside the writing statement:** an unreachable target (`Could not find server`) rolls back the user's write. `UPDATE(col)` is true when the column is in the SET list even if the value did not change. text, ntext and image columns cannot be read from `inserted`.
- A **bulk change across tables** needs no run-time metadata walk when the script is generated: write the plain `UPDATE` per table and column.
- **History by trigger or by the engine:** an `AFTER UPDATE, DELETE` trigger reads `deleted` (the before-image); `EXISTS (SELECT 1 FROM inserted)` tells update from delete. Use `SYSUTCDATETIME()` and `SUSER_SNAME()` (the login: a service account when the app shares one). A system-versioned table (`PERIOD FOR SYSTEM_TIME`, `SYSTEM_VERSIONING = ON (HISTORY_TABLE = ...)`) needs a primary key, keeps the history itself and records no user; refuse a trigger-based trail on it. History tables carry no key, default or foreign key from the source, and `nvarchar(max)` must be declared `max`, not `(0)`.

| Wrong | Right | Why |
|---|---|---|
| Making a column IDENTITY with an in-place change | Rebuild in one transaction with `SET XACT_ABORT ON`: `sp_rename` the table to `_old`, drop the foreign keys that point at it (and the child's FK to it) first, `CREATE TABLE` the new one with `IDENTITY(1,1)` and a primary key with a **different name** from the old one (the old key keeps its name after `sp_rename`; the same name fails with Msg 2714), `SET IDENTITY_INSERT ... ON`, `INSERT ... SELECT` with an explicit column list (ids kept), `OFF`, `DROP TABLE _old`, then recreate defaults, indexes and foreign keys `WITH CHECK`. Create named defaults and FKs after the old table is dropped. Make it re-runnable by testing `COLUMNPROPERTY(OBJECT_ID('dbo.T'), 'Col', 'IsIdentity')`; afterwards check `sys.foreign_keys.is_not_trusted` is 0 everywhere. Rename a column with `sp_rename 'dbo.T.Old', 'New', 'COLUMN'` (it prints a harmless "Caution") | A column cannot be made IDENTITY in place. Default, FK and PK names are schema-wide |
| `EXEC sp_executesql N'..' + @x + N'..'` | Build the statement in an `nvarchar(max)` variable first | It is a syntax error |
| `STRING_AGG` with a separator such as `N',' + @nl` | A literal or a variable | The separator must be one of those |
| An aggregate mixing an outer-query column with inner columns | Use the inner `c.object_id` in `IDENTITYPROPERTY` | Msg 8124 |
| Quoted literals built with doubled quotes by hand | `'N' + CHAR(39) + REPLACE(col, CHAR(39), CHAR(39) + CHAR(39)) + CHAR(39)` | Keeps the quoting readable and correct |

- **Generating `schema.sql` and `seed.sql`:** run `C:\Github\tools\mssql_gen_scripts.py` (see the README there) instead of rewriting a generator. Run the result with `sqlcmd -I -f 65001` (`QUOTED_IDENTIFIER` on, UTF-8 input), and send `money` and `smallmoney` out with `CONVERT(nvarchar(50), col, 2)` or the values are rounded. The catalog queries, the literal style for each column type, system-versioned tables, triggers and filtered indexes are in [reference/schema-seed-generation.md](reference/schema-seed-generation.md): read it when the tool is not enough.
- **Unique name indexes** that are not filtered (`IX_Unique_*Name`) already cover a "unique among active rows" rule more strictly but also block reusing the name of an inactive row: say so before adding a filtered index.

## Paged, searched and sorted lists through EF Core

| Wrong | Right | Why |
|---|---|---|
| A selective search plus `ORDER BY` a joined column plus `OFFSET/FETCH` left as is | Run the SQL EF logged with `SET STATISTICS TIME ON`, then with `OPTION (USE HINT('DISABLE_OPTIMIZER_ROWGOAL'))`. Fix with a `DbCommandInterceptor` that appends the hint to commands whose text starts with the tag from `.TagWith("DisableRowGoal")`; tag only queries that carry a search (a plain newest-first page needs the row goal). Measure before keeping it | It can take 4 to 17 s on 100,000 rows while each part alone takes under 100 ms: SQL Server assumes the first 10 matches come quickly (a row goal). Reproduced on `TimeEntry_test` (`E_TimeSheet`, 100,000 rows, joined to `Employee`, ordered by the employee name, page 1): a text filter that matches nothing OR-ed with a list of 3 employee ids took 17,979 ms with the default plan and 26 ms with the hint, while the text filter alone took 85 ms and the joined sort alone 18 ms. It did not reproduce on two small synthetic schemas, so it depends on the data and the exact query |
| `list.Contains(x.Col)` on a big list | `EF.Constant(list).Contains(x.Col)` for lists of a few hundred ids. Find those ids with a small first query instead of joining the big table inside the filter | In EF Core 9.0.16 it becomes `IN (SELECT [i].[value] FROM OPENJSON(@__ids_0) ...)`, which gave that bad plan on `TimeEntry_test`: 18,058 ms against 31 to 49 ms with the hint, as `sp_executesql` statements with the exact EF text. EF Core 10.0.13 sends one parameter per element (`IN (@ids1, @ids2, @ids3)`) instead, and the same query ran in 110 ms (82 ms with the hint), so the problem is mostly an EF Core 9 one. The constant form inlines `IN (1, 2, 3)` in both |
| Saving a row through EF Core on a table with an enabled trigger | Disable the trigger, or tell EF not to use `OUTPUT` | EF's default `OUTPUT` clause fails (`The target table ... cannot have any enabled triggers if the statement contains an OUTPUT clause without INTO`); the real database does the same, so a generated EF API cannot save such a table |
| A script with a view or trigger that is not first in its batch | `GO` before and after (sqlcmd reads it, ADO.NET does not) | A view or trigger must be first in its batch. `sys.sql_modules.definition` is the whole original `CREATE` text |

- To see the SQL EF sends, see "Seeing the SQL EF sends" in the `efcore-aspnet-review` skill.
- Building a page endpoint, a duplicate-row guard or a test provider swap is EF Core and ASP.NET work, not SQL Server work: see the `efcore-aspnet-review` skill.

## How this was verified

Tested on this machine. The evidence, the versions used and the items **not re-run** are in [reference/verification.md](reference/verification.md): read it before saying a claim here was tested, and treat anything listed there as not re-run as unverified.

## What this does not prove

- That a rule holds on another SQL Server version or edition, on Azure SQL, or in the Linux container image.
- That the row-goal hint or `EF.Constant` helps another query, table or data size: measure it first, as the table says.
- That a script generated from the catalog is complete: routines, linked servers, non-`dbo` schemas and other collations were not all covered.
