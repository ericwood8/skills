# Generated documents and checks

Read this when: you generate OpenAPI, Mermaid, Markdown, validators or other non-code output, or write a test for generated text.

## Generated documents and checks

- **OpenAPI:** read back with `Microsoft.OpenApi.Readers` (`diagnostic.Errors` empty, look at `Paths`/`Components.Schemas`), then probe the running API: every documented GET answers 200, `PUT /{id}` with a wrong body 400 (a missing route and a missing row are both 404), schema property names equal a real response's keys.
- **Mermaid:** `mermaid.parse(text)` under jsdom, plus a deliberately broken line so the check can fail. **Markdown tables:** escape a pipe in a cell.
- **FluentValidation and other libraries:** compile the generated classes in a scratch project with the real entities and the library. Other tools: `tsc --strict` over generated `.ts`, `Grpc.Tools` for `.proto`, the real database for SQL (SQL Server `BEGIN TRAN ... ROLLBACK`, PostgreSQL `CREATE DATABASE x TEMPLATE y`, MySQL apply and drop).
- A test column has to be one the screen shows (a text column of 100+ characters is multi-line and left out of grids); a column name with a quote breaks generated TypeScript and C#.
- Use `$(cygpath -u "$TEMP")` for Git Bash paths. Test helpers that render a template `Assert.Fail` with errors joined on one line (`ReplaceLineEndings(" ")`).
- The repository's leftover-names test fails a file that names the project or machine you tested on: say "a real server" in tests, docs and template comments.
- A probe that deletes can delete real data: check the target's children, probe a scratch copy, or capture and restore the row.
