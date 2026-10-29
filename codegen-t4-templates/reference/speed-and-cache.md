# Speed, the template cache and warnings

Read this when: generation is slow, the disk cache returns stale results, or a regenerated sample builds with warnings.

## Speed, the disk cache and generated-code warnings

- A template compile is a compiler process (about 1 s); everything else is cheap (Core calls are microseconds, a warm whole-sample generation about 1.4 s). Do not tune Core before looking at compiles and sample builds. Use `[GeneratedRegex]` rather than `RegexOptions.Compiled`.
- `TemplateCache` compiles each template once and keeps the result on disk, keyed by template text, the public surface of Core and the runtime (`CODEGENNEW_TEMPLATE_CACHE` names the folder, `off` disables). A change inside a Core method body keeps entries valid; a changed signature, constant or enum value compiles again. Tests that need a real compile use `TemplateCache.WithoutDisk()` or `InFolder(temp)`; tests that edit a template copy run it through `Repo.Cache`. Call `Prepare(paths)` early so the other cores compile while the first templates run.
- `required` plus `[SetsRequiredMembers]` does not satisfy CS8618: a NOT NULL string property gets `= "";` and a NOT NULL `byte[]` gets `= [];`. Check a regenerated sample builds with 0 warnings, not only 0 errors.
- A cache that outlives a run must not hold the last model: remove session values after each `Process()`.
