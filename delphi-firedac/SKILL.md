---
name: delphi-firedac
description: Use when a Delphi program talks to a database through FireDAC - connecting, running queries, binding parameters, reading rows, transactions - for SQLite, SQL Server, MySQL or PostgreSQL. Trigger words - TFDConnection, TFDQuery, TFDTransaction, FDConnection.Params, DriverName, ParamByName, ExecSQL, StartTransaction, TFDPhysSQLiteDriverLink, FireDAC.
---

# Delphi FireDAC: connection, queries, parameters, transactions

Written from the FireDAC documentation and the unit layout; **not compiled** (see "How this was verified"). Check every identifier against the user's installed units or the docwiki before relying on it, and say which you used. The generic Delphi rules (try/finally, `Free`, strings) are in the `delphi` skill; the SQL dialect facts are in the `mssql`, `mysql` and `postgres` skills.

## Units to put in `uses`

| Need | Unit |
|---|---|
| `TFDConnection`, `TFDQuery`, `TFDTransaction`, `TFDStoredProc`, `TFDMemTable` | `FireDAC.Comp.Client` |
| `TFDParams`, `ftString` parameter types | `FireDAC.Stan.Param` |
| `dtInt32`, `TFDDataType` | `FireDAC.Stan.Intf` |
| The driver for the database (it must be linked in, or the connection fails at run time) | one of `FireDAC.Phys.SQLite`, `FireDAC.Phys.MSSQL`, `FireDAC.Phys.MySQL`, `FireDAC.Phys.PG` |
| Console programs, no UI | `FireDAC.ConsoleUI.Wait`, and `FireDAC.Stan.Def`, `FireDAC.Stan.Async`, `FireDAC.DApt` |
| VCL programs | `FireDAC.VCLUI.Wait` (the IDE adds it when you drop a `TFDGUIxWaitCursor`) |

A missing `FireDAC.Phys.*` unit gives "[FireDAC][Phys]-... Driver not found" only when the program runs, never at build time. The Community edition ships the `.dcu` files for FireDAC but not their source, so you cannot read the component code to check a signature: use the docwiki.

## Connect

```delphi
lConn := TFDConnection.Create(nil);
try
  lConn.DriverName := 'SQLite';                 // 'MSSQL', 'MySQL', 'PG'
  lConn.Params.Values['Database'] := 'C:\data\app.db';
  lConn.LoginPrompt := False;                   // else a dialog may appear
  lConn.Connected := True;
  ...
finally
  lConn.Free;                                   // closes it
end;
```

| Database | `DriverName` | Typical params |
|---|---|---|
| SQLite | `SQLite` | `Database` (file path), optional `LockingMode=Normal` |
| SQL Server | `MSSQL` | `Server`, `Database`, `OSAuthent=Yes` or `User_Name` + `Password` |
| MySQL | `MySQL` | `Server`, `Database`, `User_Name`, `Password`, `Port`; needs the MySQL client library (`libmysql.dll`) beside the exe or on the path |
| PostgreSQL | `PG` | `Server`, `Database`, `User_Name`, `Password`, `Port`; needs `libpq.dll` |

- Passwords come from an environment variable or a prompt, never from a literal in the source, a `.dproj` or a saved connection definition in a repository.
- A named connection definition (`ConnectionDefName`) lives in `FDConnectionDefs.ini` outside the repository; keep passwords out of it too.
- Set `Params.Values[...]` before `Connected := True`; changing a parameter of an open connection does nothing until it reconnects.

## Query and read rows

```delphi
lQuery := TFDQuery.Create(nil);
try
  lQuery.Connection := lConn;
  lQuery.SQL.Text := 'select Id, Name from Customer where Active = :Active order by Name';
  lQuery.ParamByName('Active').AsBoolean := True;
  lQuery.Open;
  while not lQuery.Eof do
  begin
    Writeln(lQuery.FieldByName('Name').AsString);
    lQuery.Next;
  end;
finally
  lQuery.Free;
end;
```

- `Open` is for a statement that returns rows; `ExecSQL` is for `insert`, `update`, `delete` and DDL. Calling `Open` on an `update` raises "Cannot open a command that does not return a result set"; `ExecSQL` on a `select` discards the rows.
- `lQuery.RowsAffected` after `ExecSQL` is the count the driver reports.
- Cache a `TField` outside the loop (`lName := lQuery.FieldByName('Name')`) when reading many rows: `FieldByName` does a name lookup each call.
- `FetchOptions.Mode` and `RowsetSize` control how many rows are fetched at once; leave the defaults until a measured problem. `RecordCount` may force a full fetch, so do not call it in a loop.
- One-off scalar: `lConn.ExecSQLScalar('select count(*) from Customer')`, and `lConn.ExecSQL('delete from Log')` for a statement with no result.

## Parameters

| Wrong | Right | Why |
|---|---|---|
| `SQL.Text := 'select * from T where Name = ''' + lName + ''''` | `where Name = :Name` and `ParamByName('Name').AsString := lName` | Concatenating user text is SQL injection and breaks on a quote character |
| `ParamByName('D').Value := lDate` with an untyped value | Set `DataType` first (`ftDate`, `ftDateTime`) or use `AsDate` / `AsDateTime` | An untyped parameter takes its type from the first value and a null can leave it `ftUnknown` |
| Setting a null with `Value := ''` | `ParamByName('X').Clear` or `Value := Null` (`System.Variants`) | An empty string is a value, not null |
| A literal containing a colon in the SQL text (`'12:30'`) | Pass the value as a parameter | FireDAC scans the text for `:Name` and may read the literal as a parameter |
| A `where ... in (:List)` bound to a list | Build `:P1, :P2, ...` one parameter per element | A parameter is one value |

- FireDAC macros (`!Name`) substitute text, not values: never fill a macro from user input.
- For many rows, set the parameter arrays (`lQuery.Params.ArraySize := N`, `Params[0].AsStrings[i] := ...`) and call `lQuery.Execute(N)`: one round trip instead of N.

## Transactions

```delphi
lConn.StartTransaction;
try
  lConn.ExecSQL('update Account set Balance = Balance - 10 where Id = 1');
  lConn.ExecSQL('update Account set Balance = Balance + 10 where Id = 2');
  lConn.Commit;
except
  lConn.Rollback;
  raise;
end;
```

- By default a `TFDConnection` runs in auto-commit: each statement commits alone. `StartTransaction` suspends that until `Commit` or `Rollback`.
- Pair every `StartTransaction` with `Commit` or `Rollback` on every path; a transaction left open holds locks (on SQL Server, until the connection closes).
- `lConn.InTransaction` tells whether one is open. Nested `StartTransaction` calls are counted by FireDAC (`TxOptions`); do not rely on partial rollback inside one.
- Several queries that must share a transaction share one `TFDConnection`, or one `TFDTransaction` set as each query's `Transaction`.
- A read-only report can use `TxOptions.Isolation := xiReadCommitted` (the default) or the database's snapshot mode; set it before `StartTransaction`.

## Threads

A `TFDConnection` and the datasets on it belong to **one thread**. Give each thread its own connection (or use `TFDManager` connection pooling with `Pooled=True` in the connection definition) and open and free it inside that thread. Sharing one connection across threads gives errors that look random.

## Reporting what you could and could not check

End an answer about FireDAC with: what ran (nothing, if you could not compile), which units and driver parameters you took from the docwiki or the user's code, and which database client library the user needs installed.

## How this was verified

- Read on 2026-10-29: this machine has Delphi 12 Athens Community edition (`bds.exe` product name `Personal`). The FireDAC `.dcu` files are present under `Studio\23.0\lib\win32\release`, and only three FireDAC `.pas` files ship as source (`FireDAC.Phys.pas`, `FireDAC.Phys.SQLGenerator.pas`, `FireDAC.Phys.SQLPreprocessor.pas`), so the component classes could not be read.
- The command-line compiler refuses to run in this edition ("This version of the product does not support command line compiling"), and an MSBuild run of a small project compiled nothing. **No example here was compiled or run.**
- The docwiki returned HTTP 403 to the fetch tool, so the statements above come from the FireDAC documentation as remembered, not from a page read in this session.

## What this does not prove

- That an example compiles, or that a parameter name (`OSAuthent`, `User_Name`) is spelled as the installed driver expects.
- Behavior on other Delphi versions, other FireDAC editions or other drivers than the four listed.
