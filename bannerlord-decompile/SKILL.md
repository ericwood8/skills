---
name: bannerlord-decompile
description: Decompile a Mount & Blade II Bannerlord mod's compiled DLLs back into buildable C# source with ILSpy's CLI (ilspycmd), and get the result compiling again against the real game or BUTR reference assemblies. Use when asked to decompile a Bannerlord mod, inspect what a compiled Bannerlord mod DLL actually does, patch/recompile a Bannerlord mod without its original source, or look up a TaleWorlds.* API signature from the real installed game assemblies.
---

# Bannerlord Mod Decompiling (ILSpy / ilspycmd)

## Setup

```bash
dotnet tool install -g ilspycmd
```

If the user already has the ILSpy GUI app installed, install the matching version
(check `AppData\Local\Programs\ILSpy\ILSpy.exe`'s file version) so output is consistent.

## Decompiling

Full compilable project (one `.cs` file per type) — the default for actually recompiling:

```bash
ilspycmd "<dll>" -o "<outdir>" -p
```

For quick lookups without a full decompile:

```bash
ilspycmd "<dll>" -t "Fully.Qualified.TypeName"      # decompile just one type
ilspycmd "<dll>" --dump-table TypeDef | grep -i Foo # RELIABLE type/nested-type existence check
ilspycmd "<dll>" --dump-table TypeRef               # which external assembly a type comes from
ilspycmd "<dll>" --dump-table AssemblyRef           # every assembly + version the DLL references
```

`-l c,i,s,d,e` (list classes/interfaces/structs/delegates/enums) is faster to type but gave
false negatives in practice — when a `-l` search comes up empty and you need to be sure a type
really doesn't exist, confirm with `--dump-table TypeDef` before concluding anything (e.g. before
telling the user an API "doesn't exist in this game version").

## Getting a decompiled Bannerlord project to compile again

- **Never** look for `TaleWorlds.*` on NuGet — it's the game's own proprietary code, never
  published there. Reference the BUTR community's `Bannerlord.ReferenceAssemblies.Core` package
  instead, pinned to the actual installed game version: `Version="<gameVersion>.*-*"` (e.g.
  `1.4.8.*-*`). One "Core" package bundles nearly everything a typical mod needs — confirmed to
  include `TaleWorlds.CampaignSystem(.ViewModelCollection)`, `TaleWorlds.MountAndBlade`,
  `TaleWorlds.GauntletUI*`, `TaleWorlds.Core*`, `TaleWorlds.Library`, `TaleWorlds.TwoDimension`,
  `TaleWorlds.Localization`, `TaleWorlds.ObjectSystem`, etc. It is **not** split per game module.
- These reference assemblies are compile-time-only stubs (every method body is `throw null;`) —
  fine to compile against, useless for reading real logic. To see actual behavior, decompile the
  **real** assembly from the game's own `bin\Win64_Shipping_Client` folder instead.
- `dotnet new install Bannerlord.Templates` + `dotnet new create blmodfx --name X ...` scaffolds a
  known-good reference project fast (Harmony + reference-assemblies wiring already correct) — use
  the explicit `create` subcommand; the bare `dotnet new blmodfx ...` shorthand failed on the
  .NET 10 SDK. Likewise `dotnet new sln` on that SDK defaults to a `.slnx` file, not `.sln` —
  `dotnet sln x.slnx add <csproj>...` still works normally.
- If the mod is split across sibling DLLs (e.g. `Foo.dll` + `Foo.Core.dll`), wire a real
  `<ProjectReference>` between their two decompiled `.csproj` files. Don't leave a
  `<Reference><HintPath>` pointing at the old original DLL — edits to the sibling project would
  then be silently ignored on rebuild.
- Nullable/`init`-accessor polyfill files (`System.Diagnostics.CodeAnalysis.*`,
  `System.Runtime.CompilerServices.IsExternalInit`) sometimes appear as extra loose decompiled
  `.cs` files. Delete them before re-adding the `Nullable`/`IsExternalInit` NuGet packages for
  `<Nullable>enable</Nullable>` support, or you'll get duplicate-type errors.

## Fixing invalid C# that ilspycmd emits

ilspycmd's output is not always valid, compilable C# as-is — it has several *systematic*
decompiler artifacts. See [ARTIFACTS.md](ARTIFACTS.md) for the checklist and fix pattern for each
(ref-casts, `_002Ector`, protected-member access through a cast, unqualified nested types, and
more) before assuming a decompile error is a real bug in the mod.
