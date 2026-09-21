---
name: efcore-aspnet-review
description: Checks and known defects for EF Core entities plus ASP.NET minimal-API CRUD classes with a generic repository — a no-database harness that dumps the EF model and each entity's public surface so before/after can be diffed, and the specific bugs found (a static field never assigned, GetById that throws instead of returning null, Update without an exists or id check, a dead NoContent branch, enum-typed properties mapped to columns that do not exist). Use when generating or refactoring entity classes, repositories or API endpoint classes, when verifying that a generated entity equals a hand-written one, or when a CRUD API returns wrong 404/Location values.
---

# EF Core entities and minimal-API CRUD: verify and review

## Prove two versions of the entities are equivalent (no database needed)

Create a throwaway console project (in the scratchpad, not the repo) that references the project holding the entities and `DbContext`:

- Reference the entity project and add `<PackageReference Include="Microsoft.EntityFrameworkCore.SqlServer" Version="<same as EF Core in the project>"/>`. Without pinning the provider, building the model throws `TypeLoadException: Microsoft.EntityFrameworkCore.Metadata.Internal.AdHocMapper` (provider and EF Core versions differ through a transitive package).
- Build the context with a placeholder connection string: `new DbContextOptionsBuilder<Ctx>().UseSqlServer("Server=placeholder;Database=placeholder;Integrated Security=true").Options`. Building the model never connects.
- Write two sections to a text file: (1) for each type in the Entities/Enums namespaces, each public declared property as `required? Type Name [WriteState nullability]` (use `NullabilityInfoContext`, and the `RequiredMemberAttribute` name for `required`) plus whether `ToString` is overridden and each enum's members with values; (2) `ctx.GetService<IDesignTimeModel>().Model.ToDebugString(MetadataDebugStringOptions.ShortDefault)`.
- Run it before and after; `diff` the files. Anything in the diff must be explained: a `Required` flip in the EF model, or `Nullable` -> `NotNull` in the surface, is a tightening that can break API clients.

## What the dump revealed

- An entity property of an **enum type** next to its FK int (`Response.ResponseType` beside `ResponseTypeId`, `TimeEntryUser.SY_Role`) is mapped by EF as a *scalar column* with that name; if the table has no such column every query on that entity fails at runtime. Check model properties against the real columns (`sys.columns`).
- `[ForeignKey(nameof(X))]` naming a navigation that does not exist is tolerated by EF; do not copy that pattern.
- `required` on a navigation property forces JSON clients to send the whole related object; a nullable navigation is the safe form. `required` on a scalar entity property makes System.Text.Json refuse a body that omits it (even if the controller would fill it in afterwards).
- A collection navigation on the parent plus a reference navigation on the child means EF fix-up links them both ways and serializing the parent loops (JSON cycle). Hand-written entities often keep only one side on purpose.
- A class deriving a "name/active" base (Name + IsActive) requires the table to have exactly those columns; a table whose column is `UserName` cannot use it.

## Defects found in the API/repository pattern (check for them in similar code)

- `protected static readonly string? _apiSubDir;` declared in the base class but only ever assigned as a **local** `out` variable in `Register()` — static handlers read the field, get null, and build `Location` headers like `/api/5`. Assign it in a static constructor.
- Route constants that no longer match the registered route (`"/restricts"` vs the real `/restrictleaves`) produce wrong `Location` values and stale `.http` files.
- Repository `GetByIdAsync` that **throws** `InvalidOperationException` when nothing is found makes the handler's `row != null ? Ok : NotFound` dead code -> 500 instead of 404. Return `T?` (`FindAsync`).
- `PUT /x/{id}` handlers that never check the id: add `if (body.Id != id) return BadRequest();` and `if (!await repo.ExistsAsync(id)) return NotFound();`. Implement `ExistsAsync` with `EF.Property<int>(e, keyName)` over `_dbSet.AsNoTracking().AnyAsync(...)`, key name from `Model.FindEntityType(typeof(T)).FindPrimaryKey().Properties[0].Name` — a tracked lookup would clash with the incoming instance when the handler then sets `Entry(row).State = Modified`.
- `bool success = await repo.AddAsync(x); if (success) Created else NoContent` where `AddAsync` always returns true: dead branch, remove it (keep the check only where the repo really returns false, e.g. a duplicate-name check).
- The delete helper calling `.Result` on an async call blocks a request thread; note it, and fix it when the file is otherwise being changed.
- After changing a repository method's return type, every caller that null-checks the result stays valid; callers that assumed non-null need `?? throw`.

## Also

Ordering by a column with `GetAllOrderByDescending(Expression<Func<T, DateTime>>)` accepts only non-nullable `DateTime`; a nullable date column will not compile. `sqlcmd` reads the data and schema you need without EF (`sys.foreign_keys`, `sys.default_constraints`, `sys.columns`).

## Find entity/column mismatches before they become runtime 500s

An entity can compile, pass its tests and still fail every query because EF selects a column the table does not have (`Invalid column name 'X'`). Three real ones, found in one pass:

- an **enum-typed property** next to its `...Id` int (`ResponseType` beside `ResponseTypeId`) — EF maps it as a column;
- a **base-class property** whose column has another name (`BaseNameActiveEntity.Name` vs a table column `UserName`);
- a **typo in the database** (`RequestExpenseSheetd`) where the property is `RequestExpenseSheetId`.

How to check: in the model-dump harness, print `table|column` for every property (`et.GetProperties()`, `p.GetColumnName(StoreObjectIdentifier.Table(et.GetTableName()!, et.GetSchema()))`), dump the real `sys.tables`/`sys.columns` with `sqlcmd -h -1 -W -s "|"`, strip `\r`, and diff them with `awk` (case-insensitive; `comm` misorders names with underscores). Then confirm at runtime with read-only GETs of every route (any 500 shows the SQL error in the API log). Keyless stored-procedure result types (`DeleteTableResult`) legitimately have no table.

Fixes that leave the database alone: an enum view of an id becomes `[NotMapped] public MyEnum X => (MyEnum)XId;` (getter only, so JSON no longer demands it); a differently-named column is mapped in `OnModelCreating` with `modelBuilder.Entity<T>().Property(e => e.P).HasColumnName("Real")` (put a comment saying to remove it if the column is renamed).

Also: `await` the `spCanDelete`-style helper instead of `.Result`; and when a UI adds or edits a row, set the related object (`row.employee = list.find(...)`) because the API returns the row without its navigation properties, so a grid column showing `row.employee?.name` stays blank until a reload.
