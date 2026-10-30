---
name: delphi-types-and-units
description: Use when declaring or reading Delphi classes, records, interfaces, enums, sets or pointers, or laying out units, a .dpr and a .dproj. Fills the gaps the delphi skill leaves. Trigger words - visibility, strict private, class var, class method, record helper, class operator, virtual override abstract reintroduce, interface GUID, set of enum, PChar, dereference, uses interface implementation, .dpr, .dproj, unit initialization.
---

# Delphi: class anatomy, interfaces, units, sets, pointers

The `delphi` skill covers the language level, memory, strings, exceptions, generics and style. This one covers how types and files are put together. Examples marked "(RTL)" were read in the Delphi 12 Athens source on this machine (`Studio\23.0\source\rtl`); the rest is from the language rules and was not compiled (see "How this was verified").

## Class declaration anatomy

| Part | Write | Notes |
|---|---|---|
| Visibility | `strict private`, `private`, `strict protected`, `protected`, `public`, `published` | `private` and `protected` are visible to everything in the same **unit**; `strict` limits that to the class (and descendants for `strict protected`). Use `strict private` for fields. The RTL does (`System.Classes`) |
| Fields | `FName: string;` | Before the first method or property in a visibility section. A field after a method in the same section needs a new section header |
| Property | `property Name: string read FName write SetName;` | `read`/`write` may name a field or a method. Omit `write` for read only. `default` marks an indexed default property |
| Class method | `class function Make: TFoo; static;` | `static` means no `Self`/no class reference (RTL: `TThread.Queue ... static`). Without `static` a class method receives the class (`Self` is the class) and may be `virtual` |
| Class variable | `class var FCount: Integer;` | One per class, not per instance (RTL: `private class var` in `System.Classes`) |
| Constructor/destructor | `constructor Create; destructor Destroy; override;` | `Destroy` is always `override`. Call `inherited` first in a constructor, last in a destructor |
| Nested type | `type TKind = (a, b);` inside the class | Declared in a visibility section; used as `TFoo.TKind` outside |

Method directives, in the order the compiler wants them: `overload; virtual; abstract;` or `override; final;` then calling convention. Do not memorize an order: the compiler tells you when a directive is misplaced.

## Inheritance and virtual dispatch

| Directive | Meaning | Mistake it prevents or causes |
|---|---|---|
| `virtual` | A descendant may replace it | Without it, a descendant's same-named method **hides** the base one: a call through the base type runs the base code |
| `override` | Replaces a base `virtual`/`dynamic` method with the same signature | Compiler warning W1010 "method hides virtual method" means you forgot `override` or `reintroduce` |
| `abstract` | No body; the class (and every concrete descendant) must override it | Calling it on an instance of a class that did not override it raises "Abstract Error" at run time (RTL: `TStrings.Get` is `virtual; abstract`) |
| `reintroduce` | Deliberately hides a base member of the same name | Needed when a new overload would hide a virtual one (RTL: `TStrings.Equals ... reintroduce`) |
| `final` | No further override | |
| `inherited` | Calls the base implementation; bare `inherited;` passes the same arguments | In a constructor omit it only when the base is `TObject` and you know it does nothing |

`class of TFoo` is a metaclass: `TFooClass = class of TFoo; var C: TFooClass := TFoo; Obj := C.Create;` calls the (virtual) constructor of the real class.

## Records, helpers and operators

- A **record** is a value type: assignment copies. It can have methods, properties and `class operator`s (RTL: `TPoint`, `TSize` in `System.Types`).
- **Operator overloading** is `class operator Equal(const Lhs, Rhs: TPoint): Boolean;` and `class operator Implicit(Value: TSmallPoint): TPoint;` (both RTL). `Implicit` converts without a cast, `Explicit` needs one. Overloading `Equal` does not give `<>`: declare `NotEqual` too.
- A **helper** adds methods to an existing type: `TStringHelper = record helper for string` (RTL `System.SysUtils`). Only one helper per type is active in a given scope, the last one brought into scope by `uses` order wins and silently hides the others. Prefer a plain function or a class method when a name could clash.

## Interfaces

```delphi
type
  ILogger = interface
    ['{8A2F6C10-3B7E-4E55-9D41-2C6F1B0A7E33}']   // GUID: needed for `Supports`/`as`, not for plain assignment
    procedure Log(const AMsg: string);
  end;

  TFileLogger = class(TInterfacedObject, ILogger)
    procedure Log(const AMsg: string);
  end;
```

- Hold an interface in an interface variable and never also hold the object in a class-typed variable that you free: reference counting and `Free` fight (the lifetime rules are in the `delphi` skill).
- `Supports(Obj, ILogger, lLogger)` checks and fetches; `as` raises `EIntfCastError` on failure. Both need the GUID.
- A class lists its base class first, then interfaces: `class(TInterfacedObject, ILogger, IOther)`. It must implement every method of every interface it lists; the compiler reports each missing one (E2003/E2291 style errors).
- An interface cannot hold fields; properties on an interface must use `read`/`write` methods.

## Enums and sets

```delphi
type
  TColor = (clRed, clGreen, clBlue);
  TColors = set of TColor;
var
  lWanted: TColors := [clRed, clBlue];
begin
  if clRed in lWanted then ...
  Include(lWanted, clGreen);  Exclude(lWanted, clRed);
end;
```

- A `set of` needs an ordinal base type with at most 256 values (RTL: `TReplaceFlags = set of TReplaceFlag`).
- `Ord(clBlue)` is 2; `TColor(1)` is `clGreen` with no range check. Enums can have explicit values `(a = 1, b = 5)`, which makes `set of` and `for ... in` less useful.
- Set operators: `+` union, `-` difference, `*` intersection, `<=` subset. `[]` is the empty set.
- `{$SCOPEDENUMS ON}` makes enum members need the type name (`TColor.clRed`), which avoids name clashes between enums.

## Pointers

| Need | Write | Notes |
|---|---|---|
| Pointer type | `PInteger = ^Integer;` | RTL already has `PInteger`, `PByte`, `PChar` (`PWideChar`), `PAnsiChar` |
| Take an address | `P := @X;` | `@` on a procedure gives its address |
| Dereference | `P^ := 5;` | `P^` on a nil pointer is an access violation |
| Record field through a pointer | `PRec^.Field` or `PRec.Field` | The compiler dereferences automatically for fields |
| Untyped | `Pointer`, `GetMem`/`FreeMem`, `New`/`Dispose` | Pair every `GetMem` with `FreeMem`, every `New` with `Dispose` |
| Pointer arithmetic | `Inc(P)`, `P + 1` on `PByte`/`PChar` | Only with `{$POINTERMATH ON}` for typed pointers other than `PChar`/`PByte` types that already allow it |
| String to C-style | `PChar(S)` | Valid only while `S` is alive and unchanged; a `PChar` of an empty string is a pointer to a `#0`, not nil |

## Units, .dpr and .dproj

- A **unit** has `interface` (what others see, with its `uses`), `implementation` (hidden, with its own `uses`), optional `initialization` and `finalization`. Put a `uses` entry in `implementation` unless the interface part needs the type: this avoids circular references (error F2047 "circular unit reference").
- `initialization` runs in `uses` order at start-up, `finalization` in reverse at shutdown. Keep them small: an exception here stops the program before `begin`.
- A **.dpr** is the program source: `program Name; uses ...; begin ... end.`. For a form project the IDE rewrites its `uses` list and `Application.CreateForm` lines. Add a unit by name, with its path in `in '...'` when it is not on the search path: `Unit1 in 'src\Unit1.pas'`.
- A **.dproj** is MSBuild XML kept by the IDE: build configurations, search paths (`DCC_UnitSearchPath`), defines (`DCC_Define`), target platforms. Edit it by hand only for something the IDE cannot do, and never trust `<ProjectVersion>` to name the Delphi version (the `delphi` skill says why).
- A **package** (.dpk) groups units into a BPL. `requires` lists other packages; `contains` lists the units. Two packages must not contain the same unit.
- Search path and unit name: a unit's file name must equal the unit name (`Unit1.pas` is `unit Unit1;`); dots in names (`Vcl.Forms`) map to dotted file names.

## Checks made on this machine

- The Studio 23.0 install is the Community edition: `bds.exe` reports `ProductName Personal`, file version 29.0.51961.7529. Its compiler refuses command-line use ("This version of the product does not support command line compiling"), so nothing in this skill was compiled.
- The Athens RTL source holds `THashSet<T>` and `TObjectHashSet<T>` in `System.Generics.Collections.pas`, and `Count` there is `NativeInt`. It holds no `TOrderedDictionary`, although the Athens docwiki lists one: the class arrived in a later 12.x update or in 13, so check the installed unit before using it on 12.
- `CompilerVersion >= 36` appears 13 times in that source and `VER360` is defined, so Athens is compiler version 36. The source contains inline `var` declarations and no `if`-expression, as the version table in the `delphi` skill says. It holds no multiline `'''` string, so that row is unconfirmed.

## How this was verified

- The tables marked "(RTL)" were read in the shipped Delphi 12 Athens source (`System.Types`, `System.SysUtils`, `System.Classes`).
- Everything else is the standard language rules written from memory. It was **not compiled**: the only compiler here is the Community edition, which blocks command-line use. Treat a statement as unverified until the user's IDE accepts it, and say so.

## What this does not prove

- That a snippet compiles, or what a given compiler warning number is on the user's version.
- That any rule holds on Delphi 11 or 13 (only the 12 Athens source was read).
