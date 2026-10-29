---
name: efcore-aspnet-review
description: Use when generating or refactoring EF Core entity classes, repositories or ASP.NET minimal-API CRUD endpoint classes, when verifying that a generated entity equals a hand-written one, or when a CRUD API returns a wrong 404 or Location value. Trigger words - Invalid column name, GetById throws, SqlQueryRaw EXEC SingleAsync, CS0826, entity column mismatch, paged list endpoint, generic CRUD base, model dump diff, provider swap in tests, duplicate-row race, EF log SQL.
---

# EF Core entities and minimal-API CRUD: verify and review

## Report format and ranking

One line per finding: `file:line | what happens | cost | fix`. Rank by cost at run time, most serious first:

1. Every query on the entity fails (a column the table does not have: `Invalid column name`; an enum-typed property beside its `...Id`).
2. Wrong status or data (`GetById` throws so a missing row is a 500; `PUT` with no id check or exists check; a read that throws on NULL).
3. Wrong `Location` header, stale route constant or `.http` file (a static field never assigned).
4. Dead branches, `.Result` in a delete helper, a blank related column after an add or edit (a front-end fix: see `react-crud-gotchas` and `angular-crud-gotchas`).

A smell with no way to detect it (a grep, the model dump, a GET of every route) is not reported. Say which of those you ran.

## When information is missing

- No database reachable: run the model-dump harness below (it never connects) and report the column check as **not done**.
- No dump of the real `sys.columns`: report entity and column mismatches as **unchecked**, not as none found.
- API not running: list the defects found by reading, marked **not run**; do not state a status code you did not see.
- Default when the user did not say: the harness plus a read of every `DbSet` against a scratch copy, and say that this is what you did.

## Prove two versions of the entities are equivalent (no database needed)

Create a throwaway console project (in the scratchpad, not the repo) that references the project holding the entities and `DbContext`:

- Add `<PackageReference Include="Microsoft.EntityFrameworkCore.SqlServer" Version="<same as EF Core in the project>"/>`. Without pinning the provider, building the model throws a `TypeLoadException` (`...Internal.AdHocMapper` in one case; provider 8.0.11 with EF Core 9.0.16 gave `Method 'get_LockReleaseBehavior' ... SqlServerHistoryRepository`) or, for provider 9.0.16 with EF Core 10.0.13, a `MissingMethodException`. A provider with an older patch of the same major version works; one with a newer major pulls the core package up and also works.
- Build the context with a placeholder connection string: `new DbContextOptionsBuilder<Ctx>().UseSqlServer("Server=placeholder;Database=placeholder;Integrated Security=true").Options`. Building the model never connects.
- Write two sections to a text file: (1) for each type in the Entities/Enums namespaces, each public declared property as `required? Type Name [WriteState nullability]` (`NullabilityInfoContext`, the `RequiredMemberAttribute` name for `required`), whether `ToString` is overridden, and each enum's members with values; (2) `ctx.GetService<IDesignTimeModel>().Model.ToDebugString(MetadataDebugStringOptions.ShortDefault)`.
- Run before and after and `diff`. Explain everything in the diff: a `Required` flip in the EF model, or `Nullable` to `NotNull` in the surface, is a tightening that can break API clients.

## Entity rules

| Wrong | Right | Why |
|---|---|---|
| An entity property of an **enum type** next to its FK int (`Response.ResponseType` beside `ResponseTypeId`) | Make the enum view `[NotMapped]` (see the fixes below) | EF maps it as a *scalar column* with that name; if the table has no such column every query on the entity fails at runtime |
| `[ForeignKey(nameof(X))]` naming a navigation that does not exist | Do not copy the pattern | EF tolerates it |
| `required` on a navigation property | A nullable navigation | `required` forces JSON clients to send the whole related object |
| `required` on a scalar property that the controller fills in afterwards | Leave it non-required | System.Text.Json refuses a body that omits it, even if the controller would fill it in |
| A collection navigation on the parent plus a reference navigation on the child | Often only one side, on purpose (hand-written entities do this) | EF fix-up links both ways and serializing the parent loops (JSON cycle) |
| A class deriving a "name/active" base (Name + IsActive) over a table with a column `UserName` | Do not derive it | The base requires exactly those columns |

## Read every table, not only build the model

A model dump misses data-dependent failures. Reading each `DbSet` with `ToListAsync` (CodeGenNew: `Docs/Verification/Tools/ef_read_all.py`) finds: a relationship that "cannot target the primary key" (thrown when the model is first built, on the first `Set<T>()`), and `SqlNullValueException` where a column EF treats as required holds NULL. `HasPrincipalKey` for a foreign key to a non-primary-key column makes that column required, so it fails on real data even though the model builds; prefer no navigation for such a key. `ToDictionaryAsync` on an `IQueryable` materialises whole entities (every required column is read); select the two columns first. A `DbContext` is not thread safe: a desktop app sharing one context needs one gate for all screens.

## Find entity/column mismatches before they become runtime 500s

An entity can compile and pass tests and still fail every query because EF selects a column the table does not have (`Invalid column name 'X'`). Causes: an enum-typed property next to its `...Id` int; a base-class property whose column has another name (`BaseNameActiveEntity.Name` vs a column `UserName`); a typo in the database (`RequestExpenseSheetd` vs `RequestExpenseSheetId`).

Check: in the model-dump harness print `table|column` for every property (`et.GetProperties()`, `p.GetColumnName(StoreObjectIdentifier.Table(et.GetTableName()!, et.GetSchema()))`), dump the real `sys.tables`/`sys.columns` with `sqlcmd -h -1 -W -s "|"`, strip `\r`, and diff with `awk` (case-insensitive; `comm` misorders names with underscores). Then confirm at runtime with read-only GETs of every route (any 500 shows the SQL error in the API log). Keyless stored-procedure result types legitimately have no table.

Fixes that leave the database alone: an enum view of an id becomes `[NotMapped] public MyEnum X => (MyEnum)XId;` (getter only, so JSON no longer demands it); a differently named column is mapped in `OnModelCreating` with `modelBuilder.Entity<T>().Property(e => e.P).HasColumnName("Real")` with a comment to remove it if the column is renamed.

## Defects in the API/repository pattern (check similar code)

| Wrong | Right | Why |
|---|---|---|
| `protected static readonly string? _apiSubDir;` in the base, only ever assigned as a **local** `out` variable in `Register()` | Assign it in a static constructor | Static handlers read null and build `Location` headers like `/api/5` |
| Route constants that no longer match the registered route (`"/restricts"` vs `/restrictleaves`) | Keep them equal | Wrong `Location` values and stale `.http` files |
| A repository `GetByIdAsync` that **throws** `InvalidOperationException` when nothing is found | Return `T?` (`FindAsync`); after changing a return type, callers that assumed non-null need `?? throw` | The handler's `row != null ? Ok : NotFound` is dead code: 500 instead of 404 |
| `PUT /x/{id}` handlers that never check the id | `if (body.Id != id) return BadRequest();` and `if (!await repo.ExistsAsync(id)) return NotFound();`. Implement `ExistsAsync` as `_dbSet.AsNoTracking().AnyAsync(e => EF.Property<int>(e, keyName) == id)` with `keyName` from `Model.FindEntityType(typeof(T)).FindPrimaryKey().Properties[0].Name` | A tracked lookup would clash with the incoming instance when the handler sets `Entry(row).State = Modified` |
| `bool success = await repo.AddAsync(x); if (success) Created else NoContent` where `AddAsync` always returns true | Remove the dead branch (keep the check only where the repo really returns false, such as a duplicate-name check) | The `NoContent` branch can never run |
| A delete helper calling `.Result` | `await` it | It blocks a request thread: with 180 concurrent requests that each wait 200 ms, `.Result` took 4,174 ms and `await` 230 ms (16 pool threads) |
| `GetAllOrderByDescending(Expression<Func<T, DateTime>>)` on a nullable date | Use it only with a non-nullable `DateTime` | It accepts only a non-nullable `DateTime` |

## Concurrency, test setup and logging

| Wrong | Right | Why |
|---|---|---|
| A duplicate-row race (one row per natural key) guarded only by a pre-check | Pre-check **and** wrap the insert in `catch (DbUpdateException)`, re-querying in the catch body to confirm it was a duplicate; rethrow if it was not | Concurrent requests pass the pre-check together. A catch filter cannot `await` |
| Swapping SQL Server for InMemory in tests by removing `DbContextOptions<TContext>` only | Remove **both** registrations | `AddDbContext` registers its configuration as an additive `IDbContextOptionsConfiguration<T>`; the two collide |

### Seeing the SQL EF sends

`env "Logging__LogLevel__Microsoft.EntityFrameworkCore.Database.Command=Information" dotnet run ...` (a name with dots cannot be `export`ed in bash), then `grep -a` the log (it has binary bytes).

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
- **SQLite is not SQL Server:** `string.Contains` is case-sensitive on SQLite (`instr`) and case-insensitive under SQL Server's default collation (use the same case in the test and say why); its foreign keys are enforced, so seed parents first. A required foreign key deletes its children by default (cascade), so a test of "delete a row that is in use" needs `OnDelete(DeleteBehavior.Restrict)` on the relationship, or the delete just succeeds.
- Do not trust a search that ORs a text filter with a list of ids and sorts by a joined name: measure it on real volume (see the `mssql` skill, row goal).

## Collapsing near-identical CRUD classes into one generic base

- `CrudApi<TEntity, TInput, TOutput>` registers the five endpoints with *instance* lambdas (`(Ctx context, HttpContext http, int id) => GetByIdAsync(Call(context, http), id)`, `[FromBody] TInput` and `[AsParameters] ListQuery` as lambda parameter attributes) and holds the one set of answers (201 + Location, 400 id mismatch, 404, 422 duplicate name, 400 in use). A class overrides small virtual hooks, not whole handlers: `PrepareCreateAsync`/`PrepareUpdateAsync` return a refusal or null, `AuthorizeAsync(call, id, Read|Write)`, `ReadAsync`/`OutputAsync` for a differently shaped output, `RegisterExtras` for its own routes; `CreateAsync`/`UpdateAsync` stay overridable for a step that is really different (a parent and its children in one save, a password hash). Layer `NamedCrudApi` (name rule + GET by name) and `OwnedCrudApi` (employee scope) on top.
- Two repositories with no shared interface (`GenericRepo`, `NameActiveRepo`: same method names, different return types) are wrapped in a small `CrudStore<T>` of delegates with two `CrudStore.For(...)` overloads, not by changing the repositories.
- `Results.Ok` of a generic output needs `TOutput : class` when `PaginatedItems<T>` has that constraint. An output built only in `ReadAsync`/`OutputAsync` makes `ToOutput` unnecessary: make it virtual and throwing, not abstract.
- A literal route segment (`/employees/lookup`) wins over `/{name}` whatever the registration order; `{id:int}` and `{name}` coexist.
- Static state in the old base (a static `_apiSubDir` so static handlers could see it) disappears when handlers are instance methods that know their route.
- With Swashbuckle on generic classes and lambdas, check that `/swagger/v1/swagger.json` is 200, lists every route, and every `operationId` is unique (`WithName` collisions break the document); put that in the live test.
- Make "fix once" real: add checks for the uniform behavior (Location equals `/api/<route>/<id>`, the same 404 and 400 on every route, a problem document everywhere, no dead `if (row == null)` on a bound body).

## How this was verified

- Written from CodeGenNew work on entities, repositories and API classes (commits 2026-09 to 2026-10).
- Re-checked on 2026-10-29 against SQL Server 2022 `tempdb` with EF Core 10.0.13, using scratch tables that were dropped afterwards:
  - An enum-typed property mapped next to its `...Id` int fails every query with `Invalid column name 'ResponseType'` when the table has no such column. The `[NotMapped]` getter-only view and `HasColumnName("Note")` for a renamed column both fix it.
  - A non-nullable C# property over a column that holds NULL fails the read with "Data is Null" (the mechanism behind the `HasPrincipalKey` note; `HasPrincipalKey` itself was not tried).
  - `required` on a property makes `System.Text.Json` refuse a body that omits it (`missing required properties including: 'Name'`).
  - `SqlQueryRaw<int>("EXEC ...").SingleAsync()` throws "non-composable SQL" and `ToListAsync()` then `Single()` works.
  - `var parameters = new[] { };` is compiler error CS0826; `new string[] { }` builds.
- The repository and endpoint defects, reproduced the same day in a small in-process minimal API on in-memory SQLite (EF Core 10.0.12), defective and fixed versions side by side:
  - `GetByIdAsync` built on `SingleAsync` answers **500** for a missing id; the same endpoint on `FindAsync` answers **404**.
  - A `PUT /things/1` with a body that has `Id` 2 and no id check answers 200 and silently changes row 2; with the check it answers 400. A `PUT` for a missing row with no exists check answers 500 (a concurrency exception); with the check, 404.
  - An exists check that loads the row with a tracked `FindAsync` and then sets `Entry(body).State = Modified` answers 500 (another instance with the same key is already tracked); the `AsNoTracking().AnyAsync(...)` version answers 204.
  - `protected static readonly string? _apiSubDir;` that is never assigned gives compiler warning `CS0649` and builds `Location` as `/api/5`.
- The paged-list helper and the generic CRUD base, written from the descriptions in this skill and run in an in-process minimal API on in-memory SQLite (EF Core 10.0.12, Swashbuckle 10.2.3), 25 employee rows with ties in the sort column:
  - No page parameters answered a plain array of 25. Pages of 10 held 10, 10 and 5 rows and no id appeared on two pages even when sorting by the tied column (the key as last `ThenBy`); descending reversed the order; the count came from the filtered query; `pageSize=5000` was cut to 1000 and the answer said so.
  - `pageNumber=0`, `pageSize=0`, an unknown sort (the message listed the usable names), an unknown direction, and `pageNumber=2000000000&pageSize=1000` (the product computed in `long`) all answered 400.
  - `string.Contains` search was case-sensitive on SQLite: "Alpha" matched 5 rows and "alpha" 0.
  - Called without HTTP, `Results.Ok(x)` is an `Ok<T>` with a `Value` property and `Results.Problem(...)` is a `ProblemHttpResult` with `StatusCode` (400) and `ProblemDetails`, as the skill says.
  - The base answered 201 with `Location: /api/employees/26`, 422 for a duplicate name, 400 for an id mismatch, 404 for an unknown id on GET, PUT and DELETE, and 204 for a good PUT. With `Restrict`, deleting a parent that has a child answered 400 and inserting a child of a missing parent was refused (SQLite enforces foreign keys).
  - A literal `/lookup` beat `/{name}` even though `/{name}` was registered first, and `/{id:int}` and `/{name}` coexisted (`/7` by id, `/Alpha5` by name).
  - `swagger.json` answered 200 and every operation created with `WithName` had a unique `operationId`; two endpoints with the same `WithName` made both the endpoint and `swagger.json` answer 500.
  - `AddAsync` returning `true` always: 6 of 6 posts answered 201, so the `NoContent` branch never ran.
- The model-dump harness, built as the skill describes (a library with entities and a context on EF Core, a throwaway console project with the provider package, a placeholder connection string; 2 entities, an enum, a `required` member, an overridden `ToString`):
  - The dump printed the public surface (`required Note [NotNull]`, `Closed [Nullable]`, enum members with values, whether `ToString` is overridden), the EF model from `IDesignTimeModel` and a `table|column` list; building the model never connected.
  - After changing a nullable `Note` into a `required` one and adding a nullable `Closed`, `diff` showed exactly those: `String Note [Nullable]` became `required String Note [NotNull]`, the model line gained `Required`, and a new `Closed (DateTime?)` property and `E_Sheet|Closed` column appeared.
  - Version rules: aligned versions worked on EF Core 9.0.16 and 10.0.13; a provider with an older patch of the same major worked (9.0.10 with 9.0.16, 10.0.0 with 10.0.13); a provider with a newer major than the project's EF Core also worked (NuGet raised the core package); a provider with an older major failed (8.0.11 with core 9.0.16: `TypeLoadException`; 9.0.16 with core 10.0.13: `MissingMethodException`).
- `GetAllOrderByDescending(Expression<Func<T, DateTime>>)` called with a `DateTime?` lambda does not compile (`CS0266` and `CS1662`); the non-nullable lambda does.
- The other base-class hooks, run in a second minimal API on SQLite:
  - `AuthorizeAsync` plus an owner-scoped layer: a user listed only their own rows and the admin all 4; reading, changing or deleting another user's row answered 403; no user answered 401; a create stored the caller as the owner; a refusal from `PrepareUpdateAsync` answered 422.
  - `OutputAsync` and `ListOutputAsync` gave a detail answer with the lines and a list answer without them. An overridden `CreateAsync` saved an order and its lines in one `SaveChanges`: a bad second line (a check constraint) answered 400 and left 1 order and 2 lines from the earlier good post.
  - Two repositories with the same method names and different return types worked behind a `CrudStore<T>` of delegates with two `CrudStore.For(...)` overloads.
  - `PaginatedItems<T> where T : class` used with an unconstrained `TOutput` is `CS0452`; an abstract `ToOutput` forces a class that builds its output elsewhere to implement it (`CS0534`), while a virtual one that throws does not.
- Moved here from the `mssql` skill, with the evidence it had: against SQL Server 2022 `tempdb` with EF Core 10.0.13, the provider swap to InMemory fails with two registered providers when only `DbContextOptions<TContext>` is removed and works when `IDbContextOptionsConfiguration<TContext>` is removed too; `catch ... when (await ...)` is compiler error CS7094. The duplicate-row race pattern was not re-run.
- Not re-run: `ReadAsync` returning a differently shaped entity for every route, and the layer that scopes by employee id in a real schema.

## What this does not prove

- That a defect listed here is present in your code: these are patterns to look for, and each needs a grep, the model dump or a request to confirm.
- That the behavior holds on another EF Core version or database provider (only EF Core 10.0.13 and SQL Server were used).
