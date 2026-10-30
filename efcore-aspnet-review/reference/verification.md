# How the efcore-aspnet-review skill was verified

Evidence behind the claims in [SKILL.md](../SKILL.md). Moved out of it so that it is not read on every load.

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
