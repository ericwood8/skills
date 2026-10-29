# Named numbers and the problem answer

Read this when: a template writes a status code, exit code or database error number, or the API answers an error (409, 400, 500).

## Named numbers and the problem answer

- A status, exit code or database error number is never bare: `HttpStatus` (`<#= HttpStatus.NotFound #>` in template text, `{HttpStatus.NotFound}` in an interpolated line), `ExitCodes`, `SqlErrorCodes`, `ConstraintErrors` in Core. `HttpStatusTests` scans the templates for a bare status written into code; a new status is added to the class first. The output stays the same, so compare before and after with a worktree of the last commit and `diff -r --strip-trailing-cr`.
- Every API project has `Problems.cs` (the `Program` group): `ApiProblems.Describe(DbUpdateException)` classifies the provider's own error number into 409 (duplicate, still in use) or 400 (missing parent, too long, missing value) and names the table; an unexpected exception is a 500 with a fixed sentence, and only Development adds `cause` and `trace`. `BadHttpRequestException` must be mapped (Development throws it instead of answering 400). A delete of a row in use stays 400 with a `detail`: every front end reads that status as "in use".
- Run a PowerShell script from Git Bash with `pwsh -NoProfile -File`; Windows PowerShell 5.1 cannot load a net10 assembly. An `Invoke-WebRequest` answer of a `problem+json` type comes back as bytes: decode before `ConvertFrom-Json`.
