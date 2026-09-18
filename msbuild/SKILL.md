---
name: msbuild
description: Concrete MSBuild/.csproj gotchas learned building CodeGenNew (a WinUI3 + CLI solution on .NET 10) — the XML-comment double-hyphen trap that hits .csproj/.xaml alike, a culture-code false-positive that silently drops EmbeddedResource files into a satellite assembly, RuntimeIdentifier singular-vs-plural breaking WinAppSDK builds, a UseWindowsForms/WinUI3 conflict, .slnx vs .sln on newer SDKs, and useful `-getItem`/`-v diag` diagnostic techniques. Use when editing a .csproj/.sln(x)/.targets file, diagnosing a build error, or figuring out why a file that should be embedded/copied isn't showing up in the output.
---

# MSBuild / .csproj gotchas

## XML comments can't contain `--` — hits .csproj AND .xaml alike

Both project files and XAML files are parsed as strict XML, and standard XML forbids `--` inside a
comment (and a comment can't end in `-->` preceded by another `-`). Writing a comment the way you'd
naturally write prose — using `--` as an em-dash — reliably breaks the build:
```
error MSB4025: The project file could not be loaded. An XML comment cannot contain '--', and '-' cannot be the last character.
error WMC9997: An XML comment cannot contain '--', and '-' cannot be the last character.   (same rule, XAML compiler)
```
This is easy to reintroduce over and over across a session (it did, repeatedly) since it's a natural
writing habit, not an obviously "wrong" thing to type. Use a real em dash (—), a colon, or a comma instead
of `--` in any `<!-- ... -->` comment, in both `.csproj`/`.props`/`.targets` files and `.xaml` files.
`//` comments in `.cs` files are NOT affected (not XML) — only literal XML files.

## `EmbeddedResource` silently vanishes for filenames containing a real ISO culture code

MSBuild's `AssignCulture` task splits each filename on dots looking for a segment that matches a known
culture code (e.g. `MyResource.fr.resx` → French satellite resource). A filename like
`SP_Insert.tt.config` has "tt" sitting in the middle — which is also the ISO 639-1 code for Tatar — so
MSBuild silently reroutes that file into a `tt\` satellite resource assembly instead of the main one.
`Assembly.GetManifestResourceNames()` on the main assembly then just won't list it. **No build warning or
error is produced** — the file compiles fine, it's just quietly not where you expect it at runtime.

**Fix:**
```xml
<EmbeddedResource Include="Templates\**\*.*" WithCulture="false" />
```

**Diagnose this class of bug** with a verbose build searching for the exact tell:
```
dotnet build -v diag 2>&1 | grep "Culture of"
```
which prints e.g. `Culture of "tt" was assigned to file "..\Templates\SP_Insert.tt.config".` — instant
root cause once you know to look for it; otherwise it looks like the item was never included at all.

## `WindowsAppSDKSelfContained=true` + `SelfContained=false` needs a singular `RuntimeIdentifier`

For a WinUI3 (Windows App SDK) project where .NET's own deployment is framework-dependent
(`SelfContained=false`) but the Windows App SDK's native runtime is still bundled
(`WindowsAppSDKSelfContained=true`), a plain `dotnet build` (no explicit `-p:Platform=x64`) can fail with:
```
error : WindowsAppSDKSelfContained requires a supported Windows architecture.
```
The fix is to set a single, always-resolved RID rather than a list:
```xml
<RuntimeIdentifier>win-x64</RuntimeIdentifier>      <!-- do this -->
<!-- not: <RuntimeIdentifiers>win-x64</RuntimeIdentifiers>  (plural, publish-oriented; not resolved for a plain build without an extra switch) -->
```

Related trap: building the **solution** (`dotnet build Foo.slnx`) vs building the **project directly**
(`dotnet build Foo.App/Foo.App.csproj`) can produce different output *folder structures* for the exact
same project — e.g. `bin\x64\Debug\net10.0-windows.../win-x64\` vs `bin\Debug\net10.0-windows.../win-x64\`
— because the solution build implicitly resolves `$(Platform)` from the solution's own platform mappings
while a bare project build may not, absent an explicit `-p:Platform=x64`. If a script or note says "look
in bin\x64\..." and the exe isn't there, check the sibling `bin\Debug\...` (no `x64` segment) before
assuming something is broken.

## `UseWindowsForms=true` conflicts with a WinUI3 project's own XAML `Page` items

Turning on `UseWindowsForms` (e.g. just to use `System.Windows.Forms.FolderBrowserDialog` for its
`InitialDirectory` support) pulls in `Microsoft.NET.Sdk.WindowsDesktop`'s markup-compile targets, which
then try to process the project's existing WinUI3 XAML `Page` items as if they were WPF XAML, failing with:
```
error MC6000: Project file must include the .NET Framework assembly 'PresentationCore, PresentationFramework' in the reference list.
```
There's no clean way to keep both in the same project. If you need a real starting-directory folder
picker in an unpackaged WinUI3 app, drop to the native Win32 `SHBrowseForFolder` via P/Invoke instead of
reaching for WinForms (see the winui3 skill for the concrete pattern).

## `dotnet new sln` produces `.slnx`, not `.sln`, on newer SDKs

On .NET 10's SDK, `dotnet new sln` generates the new XML-based `.slnx` solution format by default, not the
classic `.sln`. Build commands, CI scripts, and any tooling that hardcodes a `.sln` filename need to target
the actual generated file (`dotnet build MyProject.slnx`), not assume `.sln`.

## Quick diagnostic commands worth reaching for

- **Inspect resolved items without a full build**, to check whether an `Include=` glob is actually
  matching what you think it is (item evaluation, before any target runs):
  ```
  dotnet build MyProject.csproj -getItem:EmbeddedResource
  dotnet build MyProject.csproj -getItem:None,Content
  ```
- **Trace exactly what happened to one specific file** during a real build (which task touched it, what
  metadata it was assigned, whether it got filtered out) by grepping a diagnostic-verbosity log for that
  file's name:
  ```
  dotnet build -v diag 2>&1 | grep "SomeFile.ext"
  ```
  This is slow (diagnostic verbosity is very chatty) but is the most reliable way to find *where in the
  pipeline* a file silently stopped being what you expected, rather than guessing from the `.csproj` alone.
