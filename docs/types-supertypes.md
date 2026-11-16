# Supertypes: `struct`, `enum`, and `object`

**Status**: In development, behind the `AlloyExperimentalSupertypes` flag

Proposed by rniraclefire in issue #9, as "Faux supertypes".

## Summary

Three words name the kinds of thing Alloy declares, whatever the name of
the one at hand. `x is struct`, `x is enum`, and `x is object` test a
value. `struct<T>`, `struct`, and `enum` are types for a parameter that
takes a struct or an enum itself.

## Motivation

A function that takes a struct itself, and not a value of one, has no
type to write today. It writes `unknown` and loses every check:

```alloy
export function authored_struct<T>(
    component_name: string,
    -- today:
    struct_type: unknown,
    -- wanted:
    struct_type: struct<Authored>,
    spec: { help: string, shape: Shape }
): Entity<T>
```

The same holds at run time. `x is Point` asks for one struct by name.
No test asks "is this a struct at all", which a registry, a serializer,
or a debug printer needs.

## Design

### The tests

`is` takes three more words on its right side:

```alloy
struct Point
    x: number
end

enum Color as Red, Green end

const p = new Point { x = 1 }

print(Point is struct)   --> true
print(Color is enum)     --> true
print(p is object)       --> true
print(p is struct)       --> false: `p` is a value, `Point` is the struct
print(Color.Red is enum) --> false: a variant is not the enum
```

- `x is struct` holds when `x` is a struct: the thing a `struct`
  declaration names.
- `x is enum` holds when `x` is an enum: the thing an `enum` declaration
  names.
- `x is object` holds when `x` is a value a struct made, with `new`.

`x is not struct` negates, as every `is` does.

A type the file declares under one of the three words keeps its meaning:
with `struct object ... end` in scope, `x is object` tests that struct.

### The types

- `struct<T>` is the struct whose values are `T`.
- `struct` alone is any struct.
- `enum` is any enum.

```alloy
local function origin(point_type: struct<Point>) -> Point
    return point_type.new({ x = 0 })
end

local function describe(kind: enum, made_by: struct) -> ()
    print(kind, made_by)
end

origin(Point)
describe(Color, Point)
```

A value of type `struct<T>` constructs with `.new(fields)`. `new Name {
... }` needs the name of a struct at compile time, and a parameter is no
such name.

A wrong argument reports in the words of the type:
`Expected this to be 'Struct<any>', but got 'number'`, and
`Expected this to be 'Enum', but got 'Point'`.

### Emit

A test asks the runtime for the kind of the value:

```luau
print((__alloy.kind(Point) == "struct"))
```

`__alloy.kind` reads the shape the emit already gives each declaration.
A struct and an enum index themselves. A struct is also callable, an
enum has `is`, and an object has a struct for its metatable. No
declaration gains a field, so code built before this change answers the
same way, and a module with no such test costs nothing more.

The types are aliases of the runtime:

```luau
export type Struct<T> = { read new: (fields: any) -> T }
export type Enum = { read is: (value: any) -> boolean }
```

`struct<T>` lowers to `__alloy.Struct<T>`, `struct` to
`__alloy.Struct<any>`, and `enum` to `__alloy.Enum`, on the line the
source wrote them.

### Errors

With the flag off, a test reports: `` `is struct` is not final; set
`AlloyExperimentalSupertypes = true` under `[fflags]` in alloy.toml to
use it ``. A type stays as written, and Luau reports a type it does not
know.

### The language server

A hover on a parameter reads the type as the source wrote it,
`struct<Point>`, not the alias of the runtime. The three words do not
complete after `is` yet, and `struct` and `enum` do not complete in a
type.

## What this RFC leaves out

The proposal also asked for the parts below. They are not built, and
each needs a decision first.

- **`enumitem`.** A unit variant is a string at run time, so
  `Color.Red is enumitem` cannot tell it from the string `"Red"`. The
  test needs a new shape for a unit variant, such as a frozen table, and
  that changes what every enum costs and how one prints, compares, and
  crosses a remote. That is an RFC of its own.
- **`typeof`.** The proposal has Alloy answer `typeof(Point)` with
  `"struct"`. Every Luau file must mean what it meant, and `typeof` is a
  Luau function: a library that compares `typeof(x) == "table"` would
  change its answer. `x is struct` gives the same fact with no change to
  Luau.
- **`object` as a type.** Luau has no type that says "a table whose
  metatable is some struct". `unknown` with an `is object` test covers
  the use.

## Drawbacks

Three words gain a second meaning. `struct` and `enum` open a
declaration at the start of a statement, and name a kind after `is` and
in a type. The place tells them apart, and a reader has to learn that.

The checker does not infer `T` from the struct that a call passes: the
type of a struct refers to itself, and the solver gives `unknown`. A
generic call names the type: `make<<Point>>(Point)`. A parameter with a
fixed type, `struct<Point>`, needs nothing more.

`struct<T>` types the fields of `.new` as `any`. A wrong field in that
call reports at run time, not in the editor.

`__alloy.kind` reads a shape, not a mark. A table a Luau library builds
with the same shape, one that indexes itself and is callable, answers
`struct`. A mark on each declaration fixes that at the cost of a field.

## Alternatives

- Do nothing. The parameter stays `unknown`.
- A mark on each struct and enum, `__alloy_kind`, in place of the shape
  test. It is exact, and it adds a field to every declaration, so only
  code built after the change answers.
- Generic supertypes for enums too, `enum<T>`. No use asked for it yet.
- A lint, `faux_super_type`, that reports `typeof(x) == "struct"`. With
  no `typeof` override there is nothing to lint.

## Prior Art

- TypeScript writes "a class whose instances are `T`" as
  `new (...args: any[]) => T`. `struct<T>` is that idea with the name of
  the declaration it types.
- Luau has `typeof(Point)` for the type of one value. It names one
  struct, not any struct.
- Roblox has `typeof(x) == "Instance"` beside `x:IsA("Part")`: one test
  for the kind and one for the exact class. `is struct` beside `is
  Point` is the same pair.

## Open questions

- The shape test or a mark, see Drawbacks.
- A typed `fields` parameter for `.new`, which needs a way to name the
  fields of `T` as a type.
- Whether `enumitem` is worth a new shape for unit variants.
