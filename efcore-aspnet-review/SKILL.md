---
name: efcore-aspnet-review
description: Checks and known defects for EF Core entities plus ASP.NET minimal-API CRUD classes with a generic repository — a no-database harness that dumps the EF model and each entity's public surface so before/after can be diffed, entity/column mismatches, and API/repository defects (a static field never assigned, GetById that throws, Update without an exists or id check, dead NoContent branch, SqlQueryRaw against EXEC throwing on SingleAsync, an empty implicitly-typed SqlParameter array), plus how to build and test a paged list endpoint and one generic CRUD base. Use when generating or refactoring entity classes, repositories or API endpoint classes, when verifying that a generated entity equals a hand-written one, or when a CRUD API returns wrong 404/Location values.
---

# EF Core entities and minimal-API CRUD: verify and review

## Prove two versions of the entities are equivalent (no database needed)

Create a throwaway console project (in the scratchpad, not the repo) that references the project holding the entities and `DbContext`:

- Add `<PackageReference Include="Microsoft.EntityFrameworkCore.SqlServer" Version="<same as EF Core in the project>"/>`. Without pinning the provider, building the model throws `TypeLoadException: Microsoft.EntityFrameworkCore.Metadata.Internal.AdHocMapper`.
- Build the context with a placeholder connection string: `new DbContextOptionsBuilder<Ctx>().UseSqlServer("Server=placeholder;Database=placeholder;Integrated Security=true").Options`. Building the model never connects.
- Write two sections to a text file: (1) for each type in the Entities/Enums namespaces, each public declared property as `required? Type Name [WriteState nullability]` (`NullabilityInfoContext`, the `RequiredMemberAttribute` name for `required`), whether `ToString` is overridden, and each enum's members with values; (2) `ctx.GetService<IDesignTimeModel>().Model.ToDebugString(MetadataDebugStringOptions.ShortDefault)`.
- Run before and after and `diff`. Explain everything in the diff: a `Required` flip in the EF model, or `Nullable` to `NotNull` in the surface, is a tightening that can break API clients.

## Entity rules

- An entity property of an **enum type** next to its FK int (`Response.ResponseType` beside `ResponseTypeId`) is mapped by EF as a *scalar column* with that name; if the table has no such column every query on the entity fails at runtime.
- `[ForeignKey(nameof(X))]` naming a navigation that does not exist is tolerated by EF; do not copy that pattern.
- `required` on a navigation property forces JSON clients to send the whole related object; use a nullable navigation. `required` on a scalar property makes System.Text.Json refuse a body that omits it, even if the controller would fill it in afterwards.
- A collection navigation on the parent plus a reference navigation on the child means EF fix-up links both ways and serializing the parent loops (JSON cycle). Hand-written entities often keep only one side on purpose.
- A class deriving a "name/active" base (Name + IsActive) requires exactly those columns; a table whose column is `UserName` cannot use it.

## Find entity/column mismatches before they become runtime 500s

An entity can compile and pass tests and still fail every query because EF selects a column the table does not have (`Invalid column name 'X'`). Causes: an enum-typed property next to its `...Id` int; a base-class property whose column has another name (`BaseNameActiveEntity.Name` vs a column `UserName`); a typo in the database (`RequestExpenseSheetd` vs `RequestExpenseSheetId`).

Check: in the model-dump harness print `table|column` for every property (`et.GetProperties()`, `p.GetColumnName(StoreObjectIdentifier.Table(et.GetTableName()!, et.GetSchema()))`), dump the real `sys.tables`/`sys.columns` with `sqlcmd -h -1 -W -s "|"`, strip `\r`, and diff with `awk` (case-insensitive; `comm` misorders names with underscores). Then confirm at runtime with read-only GETs of every route (any 500 shows the SQL error in the API log). Keyless stored-procedure result types legitimately have no table.

Fixes that leave the database alone: an enum view of an id becomes `[NotMapped] public MyEnum X => (MyEnum)XId;` (getter only, so JSON no longer demands it); a differently named column is mapped in `OnModelCreating` with `modelBuilder.Entity<T>().Property(e => e.P).HasColumnName("Real")` with a comment to remove it if the column is renamed.

## Defects in the API/repository pattern (check similar code)

- `protected static readonly string? _apiSubDir;` in the base but only ever assigned as a **local** `out` variable in `Register()`: static handlers read null and build `Location` headers like `/api/5`. Assign it in a static constructor.
- Route constants that no longer match the registered route (`"/restricts"` vs `/restrictleaves`) give wrong `Location` values and stale `.http` files.
- A repository `GetByIdAsync` that **throws** `InvalidOperationException` when nothing is found makes the handler's `row != null ? Ok : NotFound` dead code: 500 instead of 404. Return `T?` (`FindAsync`). After changing a return type, callers that assumed non-null need `?? throw`.
- `PUT /x/{id}` handlers that never check the id: add `if (body.Id != id) return BadRequest();` and `if (!await repo.ExistsAsync(id)) return NotFound();`. Implement `ExistsAsync` as `_dbSet.AsNoTracking().AnyAsync(e => EF.Property<int>(e, keyName) == id)` with `keyName` from `Model.FindEntityType(typeof(T)).FindPrimaryKey().Properties[0].Name`: a tracked lookup would clash with the incoming instance when the handler sets `Entry(row).State = Modified`.
- `bool success = await repo.AddAsync(x); if (success) Created else NoContent` where `AddAsync` always returns true: remove the dead branch (keep the check only where the repo really returns false, such as a duplicate-name check).
- A delete helper calling `.Result` blocks a request thread: `await` it.
- When a UI adds or edits a row, set the related object (`row.employee = list.find(...)`): the API returns the row without navigation properties, so a column showing `row.employee?.name` stays blank until a reload.
- `GetAllOrderByDescending(Expression<Func<T, DateTime>>)` accepts only a non-nullable `DateTime`.

## `SqlQueryRaw<T>` against `EXEC <stored proc>`: do not chain `.SingleAsync()`/`.FirstAsync()`

EF composes LINQ over `SqlQueryRaw<T>(...)` by wrapping the SQL in a subquery, which an `EXEC` call cannot be. Any operator that needs composition (`.SingleAsync()`, `.FirstAsync()`, `.CountAsync()`) throws at **runtime**: `'FromSql' or 'SqlQuery' was called with non-composable SQL and with a query composing over it`. Materialize first and compose in memory:

```csharp
var rows = await context.Database.SqlQueryRaw<int>("EXEC [dbo].[Foo_SearchCount] @p1", parameters).ToListAsync();
int totalCount = rows.Single();
```

A plain `SELECT ...` string can still be composed over.

## An empty parameter array must be explicitly typed

`var parameters = new[] { };` (any call site where the array can have **zero** elements, such as a generated count query with no filters) is `CS0826` (no best type). It only shows when a caller hits the zero case. Write `new SqlParameter[] { }`; grep for `new[]` in code that builds a `SqlParameter[]` from a collection that can be empty.

## A paged list endpoint: how to build and test it

- **One helper for every list**: `rows.ToResultAsync(query, sortMap, defaultSort, key, search, toDto)`. A `SortMap<T>` of name to `OrderBy` expression is the whitelist (never build a column name into an expression); the key is always the last `ThenBy` so pages never overlap; read the page `AsNoTracking()`; `Count` comes from the filtered query before `Skip/Take`; the answer carries the page size actually used. No page parameter keeps the plain-array answer so existing callers do not break.
- **Validate before the database does:** page index below 0, page size below 1, an unknown sort name or direction are 400 problem responses (the sort message lists the usable names); cap the page size; compute `(pageNumber - 1) * pageSize` in `long` before it reaches a 32 bit `OFFSET`.
- **Test the helper without HTTP on in-memory SQLite** (`SqliteConnection("DataSource=:memory:")`, `EnsureCreated`): seed rows with ties in the sort column and assert pages hold the size, no key is on two pages, descending reverses, search narrows rows and count, the cap, a bad page. `TypedResults.Ok(x)` is `Ok<T>` with a `Value` property and `Results.Problem(...)` has `StatusCode` and `ProblemDetails`: read them by reflection, no `DefaultHttpContext`.
- **SQLite is not SQL Server:** `string.Contains` is case-sensitive on SQLite (`instr`) and case-insensitive under SQL Server's default collation (use the same case in the test and say why); its foreign keys are enforced, so seed parents first.
- Do not trust a search that ORs a text filter with a list of ids and sorts by a joined name: measure it on real volume (see the `mssql` skill, row goal).

## Collapsing near-identical CRUD classes into one generic base

- `CrudApi<TEntity, TInput, TOutput>` registers the five endpoints with *instance* lambdas (`(Ctx context, HttpContext http, int id) => GetByIdAsync(Call(context, http), id)`, `[FromBody] TInput` and `[AsParameters] ListQuery` as lambda parameter attributes) and holds the one set of answers (201 + Location, 400 id mismatch, 404, 422 duplicate name, 400 in use). A class overrides small virtual hooks, not whole handlers: `PrepareCreateAsync`/`PrepareUpdateAsync` return a refusal or null, `AuthorizeAsync(call, id, Read|Write)`, `ReadAsync`/`OutputAsync` for a differently shaped output, `RegisterExtras` for its own routes; `CreateAsync`/`UpdateAsync` stay overridable for a step that is really different (a parent and its children in one save, a password hash). Layer `NamedCrudApi` (name rule + GET by name) and `OwnedCrudApi` (employee scope) on top.
- Two repositories with no shared interface (`GenericRepo`, `NameActiveRepo`: same method names, different return types) are wrapped in a small `CrudStore<T>` of delegates with two `CrudStore.For(...)` overloads, not by changing the repositories.
- `Results.Ok` of a generic output needs `TOutput : class` when `PaginatedItems<T>` has that constraint. An output built only in `ReadAsync`/`OutputAsync` makes `ToOutput` unnecessary: make it virtual and throwing, not abstract.
- A literal route segment (`/employees/lookup`) wins over `/{name}` whatever the registration order; `{id:int}` and `{name}` coexist.
- Static state in the old base (a static `_apiSubDir` so static handlers could see it) disappears when handlers are instance methods that know their route.
- With Swashbuckle on generic classes and lambdas, check that `/swagger/v1/swagger.json` is 200, lists every route, and every `operationId` is unique (`WithName` collisions break the document); put that in the live test.
- Make "fix once" real: add checks for the uniform behavior (Location equals `/api/<route>/<id>`, the same 404 and 400 on every route, a problem document everywhere, no dead `if (row == null)` on a bound body).
