---
name: self-contained-deployment
description: Use when scaffolding a C# desktop or CLI app's deployment model, deciding self-contained versus framework-dependent, or handling first-run setup and config seeding. Trigger words - no installer, copy the folder, auto-create folders, seed config from embedded defaults, embedded resources, per-user settings, Mono.TextTemplating self-contained.
---

# Self-contained deployment philosophy

The goal for every C# app built this way: **copying the built output folder to another machine is the
entire installation procedure.** No installer project, no manual "create these folders first," no setup
wizard, no separate download step for default config.

## First-run behavior (do this automatically, silently)

- **Auto-create any folder the app needs** (an output directory, a templates directory, whatever) on
  startup if it's missing. No prompt, no notification — just create it and continue.
- **Auto-seed any missing config/template file from a built-in default, but never overwrite an existing
  one.** The developer may have hand-customized that file; only fill in the gap when the file doesn't
  exist at all. The default content itself should be an **embedded resource** baked into the exe (see
  below), not a loose file the app hopes is sitting next to it — otherwise "copy the exe" isn't
  enough, and you've just moved the missing-file problem instead of solving it.
- Implementation pattern: on startup, for each seedable file, `if (!File.Exists(path)) { extract embedded
  resource to path }`. Match the embedded resource by filename suffix (`resourceName.EndsWith(fileName)`)
  rather than trying to reconstruct the exact mangled resource name — simpler and avoids ambiguity when a
  filename itself contains multiple dots (see the culture-code gotcha below).

## Embed small static assets, don't ship them as loose files

Icons and other small, never-user-edited assets should be **embedded resources** compiled into the
assembly, loaded via `Assembly.GetManifestResourceStream` at runtime — not `CopyToOutputDirectory` loose
files. This eliminates an entire class of "forgot to copy the Images folder" deployment bugs and shrinks
the footprint of what has to travel with the exe. Reserve loose on-disk files for things the developer is
to open and edit directly (templates, hand-editable config).

### MSBuild gotcha: `EmbeddedResource` items vanish for file names with a culture code

A file such as `SP_Insert.tt.config` (`tt` is the Tatar culture code) is moved silently into a satellite assembly. Add `WithCulture="false"` to the `EmbeddedResource` item. The diagnosis and the evidence are in the `msbuild` skill.

## `.NET` self-contained vs framework-dependent — check what your dependencies need

`SelfContained` (bundling the .NET runtime itself) and any UI-framework-specific "self-contained" flag
(e.g. WinUI3's `WindowsAppSDKSelfContained`, which bundles a *different* runtime — the UI framework's own
native/XAML engine) are **independent knobs**. Don't assume "fully self-contained" is always the safer or
more complete choice; check whether every dependency tolerates it.

Concrete case: a library that compiles code at runtime by shelling out to `dotnet`-hosted `csc` (e.g.
Mono.TextTemplating running T4 templates) locates that `dotnet` executable by walking a **fixed number of
parent directories up from the current runtime's own directory**. For a normal framework-dependent app,
that runtime directory is the shared framework under `Program Files\dotnet\shared\...`, and the walk
correctly lands back at `Program Files\dotnet\`. For a **self-contained** deployment, the "runtime
directory" is just the app's own output folder — the same fixed walk lands somewhere nonsensical (e.g.
`MyApp\bin\x64\dotnet`), and the subprocess spawn fails with "The system cannot find the file specified,"
even though the machine has a perfectly good .NET SDK installed. If a dependency needs to compile code or
otherwise shells out expecting a real SDK on the machine, framework-dependent deployment is the correct
choice — self-contained doesn't remove that runtime SDK dependency, it just breaks the path-resolution
that used to find it.

## Where per-user mutable settings belong

Never put a JSON/XML settings file that the app itself writes back to (last-used connection, remembered
preferences, etc.) next to the exe. The exe's own folder is a **build output directory** — it gets
overwritten wholesale by `CopyToOutputDirectory` on every rebuild, and can differ between how the app was
last built or launched (different configuration/platform/output-path combinations). A setting saved there
looks like it works during one run and then silently resets on the next rebuild or a different launch
path — a confusing, hard-to-explain "my settings don't persist" bug.

Use `%LocalAppData%\<AppName>\Settings.json` (`Environment.SpecialFolder.LocalApplicationData`) instead —
stable regardless of build configuration, untouched by rebuilds, and the standard place Windows desktop
apps keep per-user state. Reserve the exe's own folder for things that travel with
that specific build (templates, reference config, embedded-resource-seeded defaults).
