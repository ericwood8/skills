# Regression checks after a template change

Read this when: a template or Core change must be shown not to move default output, across all samples or against the last commit.

## Whole-project regression check

After a template change run `codegen generate --project <sample> -o <sample folder> --dry-run --diff` over each sample's database: every file not byte-identical (line endings ignored) is listed with its diff. Expect zero differences for an unrelated change; a wanted change shows exactly where it lands. Then run the sample's `Regenerate.sh` (generate, build, tests). `.codegen-manifest.json` in the sample root tracks files: a dropped table's old files are stale; `--delete-stale` removes those you did not edit. Slow regenerate scripts go in the background with a `done` marker and a Monitor (a foreground call is cut at two minutes); do not start another build of the same project while one runs (the CLI exe and `obj` are shared).
## Proving a template change did not move the default output

- **Byte-compare against the last commit without rebuilding or stashing:** the CLI seeds `CODEGENNEW_HOME\Templates` once from the embedded resources. Generate into folder A (current templates), then overwrite the changed `.tt` files in that seeded folder with `git show HEAD:Templates/<name>.tt` and generate into folder B with the same project; `diff -r -q A B` lists every file that moved. Run it with the new settings off for each stack touched (`--stack api --essentials`, `--stack react`) and for a project with sign-in and validation on. A difference caused by an earlier uncommitted change shows up too: read the diff before blaming the new code.
- **A scratch React app:** `npm install --no-audit --no-fund` in the generated app takes seconds; esbuild reports its postinstall as blocked, and `tsc -b`, `vitest run` and `vite` still work. Run it in the foreground (a backgrounded subshell was killed when the tool call returned). Then `npx tsc -b` and `npx vitest run` before any browser step. Vite proxies `/api` to the port in its config.
