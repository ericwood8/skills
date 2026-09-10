---
name: csharp
description: Personal C# coding conventions - a project-root GlobalUsings.cs for namespaces common across that project, defaulting to modern C# 14/.NET 10 syntax unless the project is pinned to .NET Framework/Desktop, and preferring a single well-named boolean expression over scattered conditional logic. Use when writing or reviewing C# code, scaffolding a new .csproj, or deciding how to express a conditional/boolean check.
---

# C# Conventions

## GlobalUsings.cs

Every C# project (each `.csproj`) should have one `GlobalUsings.cs` file at its project root — not the same handful of `using` statements repeated at the top of dozens of files. It should contain nothing but `global using` directives:

```csharp
global using System.Text.Json;
global using Microsoft.EntityFrameworkCore;
```

Scope it per-project, not per-solution — put a namespace in a project's `GlobalUsings.cs` only if it genuinely shows up across most files *in that project* (an EF-heavy Infrastructure project's list looks different from a controller-heavy Api project's). Don't treat it as a dumping ground for every namespace the project happens to use anywhere — that just trades explicit-but-scattered for implicit-and-unclear, losing the point of the file.

This is separate from `<ImplicitUsings>enable</ImplicitUsings>` in the `.csproj`, which the SDK already uses to auto-include a baseline set of BCL/framework namespaces (more of them for `Microsoft.NET.Sdk.Web` than plain `Microsoft.NET.Sdk`) via a generated `obj/GlobalUsings.g.cs` — leave that on, and let the hand-written `GlobalUsings.cs` cover the project-specific namespaces beyond that baseline.

## Language version

Default to modern C# — currently C# 14 (ships with .NET 10 / Visual Studio 2026). **Check the project's actual `<TargetFramework>` before assuming this applies** — a Desktop app (WPF, WinForms) or a library constrained by a dependency that hasn't moved off .NET Framework may still be pinned to something like `net48`, where newer C# syntax and APIs either aren't available or aren't worth the friction. When in doubt, read the `.csproj`'s `<TargetFramework>`/`<TargetFrameworks>` value rather than assuming — don't reach for the newest syntax in a codebase that turns out to still be net48.

When the target framework does support it, prefer current idioms over older equivalents doing the same job less clearly — primary constructors, collection expressions (`[1, 2, 3]`), target-typed `new`, switch expressions, and property patterns over the more verbose pre-C#-8-12 patterns they replace. Don't chase novelty for its own sake if an older, already-idiomatic form is just as clear — the goal is clarity, not a language-feature showcase.

## Boolean expressions

Prefer a single, well-named boolean variable over scattering the same logic across multiple nested `if` checks or throwaway intermediate booleans — a good name turns a business rule into self-documenting code instead of something a reader has to reconstruct from several branches:

```csharp
bool canStream4KContent = !isBanned && (userAge >= 18 || hasPremiumAccount);

bool isEligibleForDiscount = user is { Age: > 65, IsPremiumMember: true };
```

Property patterns (`is { Prop: condition, ... }`) are usually the clearest option once a check involves more than one property off the same object — reach for them over a chain of `&&`-joined property accesses. The point is readability, not literally minimizing line count — if collapsing a check into one expression makes it *harder* to read (too many clauses, mixed precedence that needs parentheses to untangle), it's fine to split it back into a couple of named intermediate booleans instead.
