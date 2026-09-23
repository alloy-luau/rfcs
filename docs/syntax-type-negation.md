# Type negation with `~`

**Status**: Implemented

## Summary

A type may be negated. `~number` is every value that is not a number,
and it reads in a binding, a parameter, a return, a union, and a
generic bound. The compiler lowers it to a Luau type function, so the
analyzer checks it and the editor already prints it.

## Motivation

Luau's own solver has negation. It builds one whenever a test removes a
type, and it prints it. Take an `unknown` that a test narrows:

```luau
local function f(x: unknown)
    if type(x) ~= "number" then
        local y: number = x
    end
end
```

Luau answers:

```
TypeError: Expected this to be 'number', but got '~number';
`~number` cannot be `number`
```

So a reader already meets `~number` in a hover and in a diagnostic. The
reader cannot write it. The type exists in the checker and has no
spelling in the source, which is the whole gap this closes.

The shapes that want it today:

- A value that is anything but nil, without naming the union. `~nil`
  says it once; the alternative is to list every other type.
- A generic that takes anything but one type: `<T: ~string>`.
- A parameter that must not be a table, in a function that packs a
  value for the wire.
- Narrowing a broad input and giving the narrowed value a name a
  signature can carry.

## Design

### The form

`~` before a type negates it:

```alloy
local t: ~number = "text"

local function label(v: ~nil): string
    return tostring(v)
end

type NotText = ~string
```

`~` binds tighter than `|` and `&`, so `~number | string` is
`(~number) | string`. Parentheses read the other grouping:
`~(number | string)`.

`~` takes one type operand. A second `~` is an error rather than a
double negation, because `~~number` says `number` the long way:

```
error(3.x): TypeError: `~~number` negates twice; write `number`
```

### Why `~` and not `not`

Three reasons, in order of weight.

- The analyzer prints `~number`. A reader who meets the type in a hover
  writes the same characters in the source. `not number` would give one
  idea two spellings, and the one the reader meets first is not the one
  they could write.
- `not` is the value-level operator. In `if not ready then`, `not`
  takes a value; a reader who meets it in a type position reads a
  boolean first.
- `~` is unambiguous here. A type position never holds `~=`, because
  `~=` needs a left operand and a type never stands to the left of one.

`not number` stays available as an alternative in the section below, if
the tilde proves hard to read.

### Emit

Luau has no negation syntax. Its parser rejects `~number`:

```
SyntaxError: Expected type, got '~'
```

It does have the constructor, in the type function library. A file
that writes a negation declares one type function on its first line,
beside the runtime require:

```luau
type function __neg(t) return types.negationof(t) end
```

and `~T` lowers to `__neg<T>`. The function sits in the file, not in
the std. On luau-lsp 1.69.0 a type function exported from another
module reaches the importer unchecked, so `L.neg<number>` accepted `5`.
The same function declared in the file enforces:

```luau
--!strict
type function __neg(t) return types.negationof(t) end
local a: __neg<number> = "text"   -- accepted
local b: __neg<number> = 5        -- rejected
```

```
TypeError: Expected this to be '~number', but got 'number'
```

The compiler reports that last case in its own words: "`number` does
not fit: the type asks for `~number`, so it takes anything but
`number`".

Two facts that make this work. The analyzer reduces the type function
and then prints the result as `~number`, so a diagnostic names the type
the author wrote, not the lowering. And enforcement needs strict mode,
which every Alloy project has: `alloy init` writes `.config.luau` with
`languagemode = "strict"`.

The emit stays on the source's own line, because `~T` and
`__alloy.neg<T>` both sit inside one type annotation.

### Where it applies

Every type position: a binding, a parameter, a return, a field, a type
alias, a union member, an intersection member, a generic argument, and
a generic bound (`<T: ~nil>`).

`types.negationof` negates a primitive, a singleton, a class, and a
union of them. It fails on a table type and on a function type, so the
compiler reports those operands before the analyzer runs: a table
literal, an array, a std table type, a struct, and a function type.
An alias of a record reaches the analyzer, whose report the compiler
rewrites to the same words.

Two positions reject it, each with its own report:

- A wire type. A remote packs a layout off its parameters, and a
  negation names no layout. The report says to name the types the
  remote carries.
- A struct field's declared type, when the struct derives a serializer,
  for the same reason.

### The language server

- A hover on a negated type reads `~number`, which is what the analyzer
  already returns, so no mapping is needed.
- Completion after `~` offers the types in scope.
- The inlay hint for a binding whose type the analyzer narrowed to a
  negation prints `~number`, so the hint and the source agree.
- The quick fix for `~~T` rewrites it to `T`.

## Drawbacks

- One more meaning for `~`, which is half of `~=`. The positions do not
  overlap, and a reader still has to hold both.
- A negation is easy to write and hard to satisfy. `~nil` is useful;
  `~string` in a parameter tells a caller what the function will not
  take and nothing about what it will. The language cannot stop that,
  and the docs should teach `~nil` first.
- The lowering depends on `types.negationof`, which is a type function
  library detail rather than a language guarantee. If a future Luau
  removes it the lowering breaks, though a future Luau that removes it
  most likely gained the syntax instead.
- Enforcement is strict-mode only. A file that sets nonstrict gets the
  annotation and no check.

## Alternatives

- `not number`. It reads as English and needs no new symbol, and it
  gives the reader two spellings for the type the analyzer prints. Take
  it only if the tilde proves hard to read in practice.
- Both spellings, with `not T` as sugar. Two ways to write one type, and
  the formatter would have to pick one anyway.
- `Not<T>` as a std type. It needs no syntax at all and it is what the
  lowering already is. It reads as a library type rather than a type
  operator, and it does not match what the analyzer prints.
- Nothing. The type stays in the checker with no spelling, and a reader
  who meets `~number` in a hover has no way to write it down.

## Prior Art

- Luau: the solver builds and prints negation types; the parser has no
  syntax for them. This proposal gives the existing type a spelling.
- TypeScript: no negation type. `Exclude<T, U>` removes members of a
  union, which is the same idea narrowed to unions.
- Flow: `$Diff` and friends, again by union subtraction.
- Set theory and several ML dialects: `~` or a complement operator with
  the precedence used here, tighter than union and intersection.

## Unresolved questions

- Whether `~nil` should be the taught spelling for a value that is not
  nil, beside the existing rule that `T` is already non-optional and
  `T?` is the optional one.
- Whether a negation may name a struct (`~Vec2`), which is sound for
  the checker and rarely what an author means.
- Whether the compiler should simplify a negation of a union member
  against the union it sits in, so `(number | string) & ~number` reads
  as `string` in a hover rather than as the intersection.
- Whether the nonstrict gap deserves a warning when a file both sets
  nonstrict and writes a negation.
