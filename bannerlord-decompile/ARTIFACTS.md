# ilspycmd decompiler artifacts checklist

Recurring patterns where ilspycmd's decompiled output is invalid or non-idiomatic C# and needs a
manual fix before the project will compile. All confirmed firsthand decompiling a real Bannerlord
mod. Work through this list before assuming a compile error reveals an actual bug in the mod.

## `((SomeType)(ref localVar)).Method(...)`

An invalid pseudo-cast-with-ref expression ilspycmd emits when it can't cleanly express a call to
an instance method (often on a `readonly struct`) through a ref. This never compiles.

**Fix**: drop the cast entirely — `localVar.Method(...)`.

```csharp
// before (invalid)
((Rectangle2D)(ref val)).CalculateMatrixFrame(ref base.AreaRect);
// after
val.CalculateMatrixFrame(ref base.AreaRect);
```

## `obj._002Ector(args)`

`_002Ector` is the hex-escaped mangling of `.ctor` — ilspycmd failing to express a base/this
constructor-chaining call as real C# syntax. Shows up as `base._002Ector(args)`,
`((BaseType)this)._002Ector(args)`, or similar.

**Fix**: move it into a real constructor initializer on the signature line, and delete the
statement from the body.

```csharp
// before (invalid)
public ClanScreenMixin(ClanManagementVM vm)
{
    base._002Ector(vm);
    ...
}
// after
public ClanScreenMixin(ClanManagementVM vm)
    : base(vm)
{
    ...
}
```

## `((BaseType)this).SomeProtectedMember`

Accessing an inherited *protected* member through an explicit cast to the base type doesn't
compile in C# — protected access requires the qualifier to be the derived type, `base`, or
unqualified. ilspycmd emits casts like this for both protected methods and protected fields.

**Fix**: `base.Method(...)` for methods; the bare unqualified name for fields/properties accessed
from within the derived class body.

```csharp
// before (invalid)
((MBSubModuleBase)this).OnSubModuleLoad();
float scaleToUse = ((Widget)this)._scaleToUse;
// after
base.OnSubModuleLoad();
float scaleToUse = _scaleToUse;
```

## Unqualified nested types

A type nested inside another class gets referenced by its bare name only, even though it needs
the enclosing type as a qualifier to resolve outside of ilspycmd's own internal symbol table.
Shows up as `CS0246: type or namespace name 'X' could not be found`.

Examples seen: `TradeAgreement` → `TradeAgreementsCampaignBehavior.TradeAgreement`;
`DeclareWarDetail` → `DeclareWarAction.DeclareWarDetail`; `MakePeaceDetail` →
`MakePeaceAction.MakePeaceDetail`; `TooltipPropertyFlags` → `TooltipProperty.TooltipPropertyFlags`;
`DebugColor` → `Debug.DebugColor`.

**How to find the right qualifier**: decompile a related class you already know references the
type (e.g. an event-raising class whose delegate signature names it) with
`ilspycmd <dll> -t <ThatClass>` and read the fully-qualified form there. `CampaignEvents`' event
delegate signatures are a reliable place to find the real nested-type path for
action/behavior-related types.

## Real signature mismatches (not just cosmetic)

Occasionally the decompiled call site is simply wrong, not just malformed — e.g. a `ref` argument
where the compiler demands `out` (`CS1620`). When a fix from this list doesn't apply, decompile the
real interface/method with `-t` and check its actual parameter modifiers rather than assuming the
original call site was correct.

```csharp
// interface (ground truth, via -t on the real assembly)
bool HasTradeAgreement(Kingdom kingdom, Kingdom other, out TradeAgreementsCampaignBehavior.TradeAgreement tradeAgreement);
// before (invalid)
campaignBehavior.HasTradeAgreement(val, val2, ref agreement)
// after
campaignBehavior.HasTradeAgreement(val, val2, out agreement)
```
