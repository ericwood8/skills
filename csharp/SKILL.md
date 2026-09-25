---
name: csharp
description: Personal C# coding conventions - a project-root GlobalUsings.cs for namespaces common across that project, defaulting to modern C# 14/.NET 10 syntax unless the project is pinned to .NET Framework/Desktop, preferring a single well-named boolean expression over scattered conditional logic, required/init properties over mutable setters, records for DTOs, propagating CancellationToken through async call chains, and preferring extension methods over static helpers when there's one obvious subject parameter (for IntelliSense discoverability). Use when writing or reviewing C# code, scaffolding a new .csproj, designing a data model or DTO, writing an async method, deciding how to express a conditional/boolean check, or deciding whether a static helper method should be an extension method.
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

## Object construction: `required`/`init` over mutable setters

When a type has properties that must be set at construction and shouldn't change afterward, use `required` + `init` instead of a mutable auto-property or a constructor overload juggling optional parameters — the compiler then enforces that every required value is actually supplied, and the object can't drift out of a valid state after creation:

```csharp
public class User
{
    public required string Email { get; init; }
    public required Guid TenantId { get; init; }
}
```

This is about construction-time safety, not a blanket "make everything immutable" rule — a type that legitimately needs to mutate after creation (a view model tracking UI-editable state, an entity EF Core updates) should keep ordinary settable properties.

## Records for DTOs

Default to `record` (or `record struct` for small, frequently-allocated ones) for types that exist purely to carry data — API request/response shapes, EF Core projection targets, message payloads — rather than a plain `class`. Value-based equality and a concise positional-or-init syntax come for free, and there's rarely a reason a DTO needs reference identity or mutability:

```csharp
public record UserSummary(Guid Id, string Email, DateTimeOffset CreatedAt);
```

Keep using a plain `class` for anything with actual behavior (methods that do real work, EF Core entities that need identity semantics tied to a database key, not just structural equality) — the point is picking the type that matches what the type *is*, not defaulting to `record` everywhere.

## Propagate `CancellationToken` through async call chains

An async method that calls other async methods should accept a `CancellationToken` parameter and pass it down the chain, not swallow it at the top or omit it because "it's just an internal helper":

```csharp
public async Task<User> GetUserAsync(Guid id, CancellationToken cancellationToken)
{
    return await _dbContext.Users.FirstAsync(u => u.Id == id, cancellationToken);
}
```

This is easy to forget because the code compiles fine without it — a dropped token doesn't cause an error, it just means a caller's cancellation (request aborted, timeout, user navigated away) silently stops propagating partway down the stack and the expensive operation keeps running anyway. Default to threading it through; the exception is a genuinely fire-and-forget background operation that's supposed to outlive the caller's request.

## Prefer extension methods over static helpers with an obvious subject

When a static helper method has one parameter that's clearly "the thing being acted on," write it as an extension method on that type instead of a plain static method taking it as an argument. The author prefers this specifically for IntelliSense: typing `thing.` surfaces `thing.DoSomething()`, but a static `Helper.DoSomething(thing)` call is invisible until you already know the helper class exists and go looking for it.

```csharp
// Preferred -- discoverable by typing "name."
public static class StringExtensions
{
    public static bool IsNameBad(this string name) => ...;
}
if (name.IsNameBad()) ...

// Avoid when the extension-method form above reads equally naturally
public static class StringHelpers
{
    public static bool IsNameBad(string name) => ...;
}
if (StringHelpers.IsNameBad(name)) ...
```

**Don't force it.** Only make something an extension method when the method genuinely reads as an operation *on* that one parameter. Skip it when there's no single obvious subject — several equally-important inputs (a diff between two objects of the same type, where neither is more "the subject" than the other), a static factory building a new instance from a primitive (`TemplateConfig.Load(path)`: `path` is just a generic `string`, not conceptually a `TemplateConfig`, so a `path.LoadAsTemplateConfig()` extension would be a non-obvious method to find hanging off every string in the codebase), or a method that's really about the *type* itself rather than a particular instance. Test: "if I were about to type `parameterName.`, would I actually expect this method to show up there?" If yes, make it an extension method. If the honest answer is "not really, I'd go looking for a helper class instead," leave it a plain static method — a forced `this` parameter makes the subject arbitrary and can read *less* clearly than the plain static call.
