---
name: efcore-aspnet-review
description: Checks and known defects for EF Core entities plus ASP.NET minimal-API CRUD classes with a generic repository — a no-database harness that dumps the EF model and each entity's public surface so before/after can be diffed, and the specific bugs found (a static field never assigned, GetById that throws instead of returning null, Update without an exists or id check, a dead NoContent branch, enum-typed properties mapped to columns that do not exist, SqlQueryRaw against an EXEC call throwing on composing operators like SingleAsync, an implicitly-typed empty SqlParameter array failing to compile). Use when generating or refactoring entity classes, repositories or API endpoint classes, when verifying that a generated entity equals a hand-written one, or when a CRUD API returns wrong 404/Location values.
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

## `SqlQueryRaw<T>` against an `EXEC <stored proc>` call: don't chain `.SingleAsync()`/`.FirstAsync()` etc. on it

EF Core normally lets you compose further LINQ on top of `SqlQueryRaw<T>(...)` by wrapping the raw SQL in a
derived-table subquery — but a stored-procedure call (`EXEC dbo.Foo_SearchCount @p1`) can't be wrapped that
way, so any operator that needs EF to *compose* the SQL (`.SingleAsync()`, `.FirstAsync()`, `.CountAsync()`,
etc.) throws at runtime, not compile time:
```
System.InvalidOperationException: 'FromSql' or 'SqlQuery' was called with non-composable SQL and with a
query composing over it. Consider calling 'AsEnumerable' after the method to perform the composition on
the client side.
```
Fix: materialize the whole result set first with `.ToListAsync()` (which just executes the `EXEC` as-is, no
composition needed), then do the composing operator in memory:
```csharp
var rows = await context.Database.SqlQueryRaw<int>("EXEC [dbo].[Foo_SearchCount] @p1", parameters).ToListAsync();
int totalCount = rows.Single();
```
This applies to any `SqlQueryRaw`/`FromSqlRaw` call whose SQL text starts with `EXEC` rather than `SELECT`
— a plain `SELECT ...` string can still be composed over safely; a stored-procedure call cannot.

## An empty parameter array for a `SqlQueryRaw`/`SqlParameter[]` call must be explicitly typed

`var parameters = new[] { };` (or any call site where the array can end up with **zero** elements — e.g. a
generated "count" query with no filter parameters at all) is a C# compile error, `CS0826` ("no best type
found for implicitly-typed array"), because the compiler can't infer an element type from zero elements.
This only shows up once a caller of the pattern actually hits the zero-element case — a template or helper
that always emits at least one element in every sample it was tested against will look fine right up until
a table/case with no parameters exercises it. Fix: give the array an explicit element type instead of `var`:
```csharp
var countParameters = new SqlParameter[] { }; // compiles even with zero elements
```
Worth grep-ing for `new[]` in any code (hand-written or generated) that builds a `SqlParameter[]`/similar
array from a collection that could legitimately be empty.

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

## A paged list endpoint: how to build it and how to test it (2026-10)

- **One helper for every list**: `rows.ToResultAsync(query, sortMap, defaultSort, key, search, toDto)`. A `SortMap<T>` of name -> `OrderBy` expressions is the whitelist (never build a column name into an expression), the key is always the last `ThenBy` so pages never overlap, the page is read `AsNoTracking()`, `Count` comes from the filtered query before `Skip/Take`, and the answer carries the page size actually used. No page parameter keeps the old plain-array answer so existing callers do not break.
- **Validate before the database does**: page index below 0, page size below 1, an unknown sort name or direction are 400 problem responses (the sort message lists the usable names); cap the page size; compute `(pageNumber - 1) * pageSize` in `long` before it reaches a 32 bit `OFFSET`.
- **Test the helper without HTTP on in-memory SQLite** (`SqliteConnection("DataSource=:memory:")`, `EnsureCreated`): seed rows with ties in the sort column, then assert pages hold the size, no key is on two pages, descending reverses, search narrows rows and count, the cap, a bad page. `TypedResults.Ok(x)` is `Ok<T>` with a `Value` property and `Results.Problem(...)` has `StatusCode` and `ProblemDetails`: read them by reflection, no `DefaultHttpContext` needed.
- **SQLite is not SQL Server**: `string.Contains` is case-sensitive on SQLite (it becomes `instr`) and case-insensitive under SQL Server's default collation; use the same case in the test and say why in a comment. Its foreign keys are enforced, so seed parents first.
- **Do not take a generated or hand-written search that ORs a text filter with a list of ids and sorts by a joined name on trust**: measure it on the real volume (see the `mssql` skill, row goal).

## Collapsing near-identical minimal-API CRUD classes into one generic base (TimeEntryServer E04, 2026-10)

- **Shape that worked:** `CrudApi<TEntity, TInput, TOutput>` registers the five endpoints with *instance* lambdas (`(TimeEntryContext context, HttpContext http, int id) => GetByIdAsync(Call(context, http), id)`, `[FromBody] TInput` and `[AsParameters] ListQuery` as lambda parameter attributes) and holds the one set of answers (201 + Location, 400 id mismatch, 404, 422 duplicate name, 400 in use). A class overrides small virtual hooks, not whole handlers: `PrepareCreateAsync` / `PrepareUpdateAsync` return a refusal or null, `AuthorizeAsync(call, id, Read|Write)`, `ReadAsync` / `OutputAsync` for a differently shaped output, `RegisterExtras` for routes of its own; `CreateAsync` / `UpdateAsync` stay overridable for a step that is really different (a parent and its children in one save, a password hash). Layer `NamedCrudApi` (name rule + GET by name) and `OwnedCrudApi` (employee scope) on top. 15 classes of 1,951 lines became 668 plus about 300 shared.
- **Two repositories with no shared interface** (`GenericRepo`, `NameActiveRepo`, same method names, different return types) are wrapped in a small `CrudStore<T>` of delegates, with two `CrudStore.For(...)` overloads, rather than changing the repositories.
- **`Results.Ok` of a generic output needs `TOutput : class`** when the page type `PaginatedItems<T>` has that constraint. An output built only in `ReadAsync`/`OutputAsync` makes `ToOutput` unnecessary: make it virtual and throwing, not abstract.
- **Route precedence:** a literal segment (`/employees/lookup`) wins over `/{name}` whatever the registration order; `{id:int}` and `{name}` coexist.
- **Static state in the old base** (a static `_apiSubDir` set by a static constructor so static handlers could see it) disappears when handlers are instance methods that know their route.
- **Swashbuckle on generic classes and lambdas:** check that the document still builds (`/swagger/v1/swagger.json` is 200), lists every route, and that every `operationId` is unique (`WithName` collisions break the document); put that in the live test.
- **Make "fix once" real:** the old classes answered an empty 404 here and a problem document there, built Location headers three ways (`/api/projectTasks/1`, `/api/projects/1`, `/api{_apiSubDir}/1`) and had dead `if (row == null)` checks on a bound body. Add checks for the uniform behavior (Location equals `/api/<route>/<id>`, the same 404 / 400 on every route).
