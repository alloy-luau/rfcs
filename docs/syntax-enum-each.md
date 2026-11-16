# `$each`: the items of an enum

**Status**: In development, behind the `AlloyExperimentalEnumEach` flag

Proposed by rniraclefire in issue #8.

## Summary

`$each(Enum)` is an `Array` of the items of an enum, in the order the
enum declares them. A unit variant is its value, and a variant with a
payload is its constructor.

## Motivation

Code that walks every variant of an enum writes the list a second time:

```alloy
enum Phase as Load, Init, Start end

const phases = [ Phase.Load, Phase.Init, Phase.Start ]

for _, phase in phases do
    run(phase)
end
```

The list and the enum then drift. A fourth variant compiles, and the
loop never runs it. No error names the missing item, since the list is a
plain array.

The compiler already holds the variants of every enum a file declares or
imports: an exhaustive `match` reads them. `$each` gives the same list
to the program.

## Design

`$each` is an intrinsic. It takes one argument, the name of an enum.

```alloy
enum ComponentValue
    Tag
    Value(unknown)
    Struct({ [string]: unknown })
end

for index, item in $each(ComponentValue) do
    print(index, item)
end
```

The value is an `Array`, so the methods of one apply:
`$each(Phase):map(describe)`.

Each item is what `Enum.Name` reads as:

- A unit variant is its value. `$each(Phase)` holds `Phase.Load`,
  `Phase.Init`, and `Phase.Start`, and its type is `Phase[]`.
- A variant with a payload is its constructor, a function. Its item in
  the list is `ComponentValue.Value`, which builds a value when called.

The order is the order of the declaration.

The argument is a name the compiler knows as an enum: one the file
declares, or one it imports. A dotted path to an enum of a star import
or a namespace works the same way. The argument is not an expression: a
local that holds an enum has no list the compiler can read.

### Emit

The call becomes one array literal on its own line:

```luau
for index, item in __alloy.Array.from({ ComponentValue.Tag, ComponentValue.Value, ComponentValue.Struct }) do
```

Nothing is added to the enum, so an enum that no `$each` names costs
nothing more than today.

### Errors

- An argument that is no enum: `` `$each` takes the name of an enum;
  `print` is not one this file declares or imports ``.
- With the flag off: `` `$each` is not final; set
  `AlloyExperimentalEnumEach = true` under `[fflags]` in alloy.toml to
  use it ``.

### The language server

`$each` completes after `$` with the other intrinsics, and
`alloy doc $each` documents it. The type of the call is the type Luau
gives the array literal, so a wrong use reports as any wrong array does.

## Drawbacks

One more intrinsic to learn. The name `each` reads like a loop, and the
call is a list.

A variant with a payload gives its constructor, not a value. A reader
who expects every item to be a value of the enum has to learn that. The
list of a mixed enum has a union type that is wide.

## Alternatives

- The name `$items`. It says what the call returns. `$each` is the name
  of the proposal, and it reads well in the `for` that most uses write.
- A function of the std, `Enum.items(Phase)`. It needs the list at run
  time, so every enum then carries one, used or not.
- An `ITEMS` field on every enum. The same cost, and a name that a
  variant called `ITEMS` collides with.
- Constants or static fields in an `impl`, so a project writes the list
  once beside the enum. The list still drifts from the declaration. The
  static members RFC covers that feature on its own.

With nothing done, the list stays written by hand.

## Prior Art

- Rust has no built-in list. The `strum` crate derives `EnumIter`, and
  a variant with a payload takes its default payload there.
- Swift has `CaseIterable` and `allCases`, for an enum with no payload.
- TypeScript reads `Object.values` of an enum at run time.
- Roblox has `Enum.Material:GetEnumItems()`, a method on the enum.

Alloy differs in one point: the list is written at compile time, at the
place that asks for it.

## Open questions

- Should a variant with a payload be left out, so the type stays
  `Enum[]`? The first build keeps every variant, as the proposal shows.
- Should the argument take a type alias of an enum?
