---
name: self-contained-deployment
description: Personal deployment philosophy for C# desktop/CLI apps — no installer, copy-the-folder-and-go, auto-create missing folders silently, auto-seed missing config/template files from embedded defaults without ever overwriting a customized one, embed small static assets (icons, etc.) as resources rather than loose files, and where per-user mutable settings actually belong. Includes a concrete .NET/MSBuild gotcha (Mono.TextTemplating breaking under self-contained deployment, and a culture-code false-positive on double-extension filenames). Use when scaffolding a new C# app's deployment model, deciding self-contained vs framework-dependent, or handling first-run setup/config seeding.
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
  below), not a loose file the app hopes is sitting next to it — otherwise "copy the exe" isn't actually
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
actually meant to open and edit directly (templates, hand-editable config).

### MSBuild gotcha: `EmbeddedResource` items silently vanish for filenames with a real ISO culture code

MSBuild's `AssignCulture` task inspects each filename segment between dots looking for something that
matches a known culture code (e.g. `MyResource.fr.resx` → French satellite resource). If a filename
happens to contain a segment that collides with a real culture code — e.g. `SP_Insert.tt.config`, where
"tt" is also the ISO code for Tatar — MSBuild silently reroutes that file into a `tt\` satellite resource
assembly instead of the main one. `Assembly.GetManifestResourceNames()` on the main assembly then simply
won't list it, with no build warning or error at all.

**Fix:** add `WithCulture="false"` to the `EmbeddedResource` item(s):
```xml
<EmbeddedResource Include="Templates\**\*.*" WithCulture="false" />
```
Diagnose this class of bug by running `dotnet build -v diag` and grepping for `"Culture of ... was assigned
to file"` — that's the exact tell.

## `.NET` self-contained vs framework-dependent — check what your dependencies actually need

`SelfContained` (bundling the .NET runtime itself) and any UI-framework-specific "self-contained" flag
(e.g. WinUI3's `WindowsAppSDKSelfContained`, which bundles a *different* runtime — the UI framework's own
native/XAML engine) are **independent knobs**. Don't assume "fully self-contained" is always the safer or
more complete choice; check whether every dependency actually tolerates it.

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

## Where per-user mutable settings actually belong

Never put a JSON/XML settings file that the app itself writes back to (last-used connection, remembered
preferences, etc.) next to the exe. The exe's own folder is a **build output directory** — it gets
overwritten wholesale by `CopyToOutputDirectory` on every rebuild, and can differ between how the app was
last built or launched (different configuration/platform/output-path combinations). A setting saved there
looks like it works during one run and then silently resets on the next rebuild or a different launch
path — a confusing, hard-to-explain "my settings don't persist" bug.

Use `%LocalAppData%\<AppName>\Settings.json` (`Environment.SpecialFolder.LocalApplicationData`) instead —
stable regardless of build configuration, untouched by rebuilds, and the standard place Windows desktop
apps keep per-user state. Reserve the exe's own folder for things that are genuinely meant to travel with
that specific build (templates, reference config, embedded-resource-seeded defaults).
