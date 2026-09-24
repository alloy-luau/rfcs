# Parameter destructuring

**Status**: Implemented

## Summary

A parameter may be a table pattern instead of a name: `function
draw({ x, y }: Point)` binds `x` and `y` from the argument. A field may
carry its own type, `function foo({ bar: string, baz: number })`, and
the pattern then states the parameter's type. `...rest` takes the
fields the pattern does not name.

## Motivation

A function that takes an options table reads its fields on the first
line of every body:

```alloy
local function spawn(options: SpawnOptions)
    local model, at, owner = options.model, options.at, options.owner
    ...
end
```

The names the body uses are three lines from the signature. A reader of
the call site sees `spawn({ model = m, at = p, owner = plr })` and a
reader of the signature sees one word, `options`.

Alloy already destructures a binding and a loop:

```alloy
local { name, hp } = player
for _, { x, y } in points do
```

The parameter list is the position where it is missing.

## Design

### The form

A parameter is a name, or a table pattern:

```alloy
local function draw({ x, y }: Point)
    print(x, y)
end

draw(origin)
```

The pattern takes the same entries a `local` pattern takes:

- `name`, which binds the field of that name
- `name = alias`, which binds that field under another name
- `...rest`, which binds a table of the fields the pattern does not name

and one entry a `local` pattern has no use for:

- `name: Type`, which binds the field and states its type

### The type of the parameter

Two spellings, and the parameter's type comes out the same:

```alloy
-- The type is named, and the pattern only picks names out of it.
local function draw({ x, y }: Point) end

-- No name exists for the shape, so the pattern states it.
local function foo({ bar: string, baz: number }) end
```

The second is sugar for `(arg: { bar: string, baz: number })`. A
pattern that carries no field type and no annotation is an error: the
compiler will not guess the shape.

```
error(3.x): TypeError: `{ bar, baz }` has no type; annotate the
parameter, `{ bar, baz }: Options`, or type each field
```

A pattern may not mix a field type and an annotation. One of them
states the shape:

```
error(3.x): TypeError: `{ bar: string }: Options` states the shape
twice; drop the field types or drop the annotation
```

### `...rest`

`...rest` binds the fields the pattern does not name, as a table:

```alloy
local function tag({ id, ...rest }: Attributes)
    -- id is a string; rest holds every other field of Attributes.
end
```

Its type is the annotation's index type when the annotation has one
(`{ [string]: T }` gives `rest: { [string]: T }`). A pattern that
states its own field types gives `rest: { [string]: unknown }`, and the
author narrows it where it is read. `...rest` comes last; a name after
it is an error.

This `...` is not Luau's varargs. `function f(...: number)` still means
the argument list, and a pattern's `...rest` means the rest of one
table. The two may stand in one signature:

```alloy
local function log({ level }: Options, ...: string) end
```

### Emit

Luau has no parameter destructuring, so the emit takes a parameter and
opens it on the same line. The emit stays on the source's own lines:

```alloy
local function draw({ x, y }: Point)
    print(x, y)
end
```

```luau
local function draw(_p1: Point) local x, y = _p1.x, _p1.y
    print(x, y)
end
```

`...rest` emits the table-copy the array form already emits for its
tail, minus the named keys.

A struct argument keeps its metatable: the emit reads fields off the
value and never rebuilds it.

### Where it applies

Every parameter list: a `local function`, a function expression, a
method in an `impl`, a `macro`, and a `declare` block's signature. A
`remote` declaration takes no pattern, because the wire layout reads the
parameter names; the error says so.

`self` is never a pattern.

### Errors

- A pattern with no type: the first error above.
- A pattern that states the shape twice: the second error above.
- A field the annotation does not carry: `` `Point` has no field
  `z` ``, on the field's name.
- A name after `...rest`: `` `...rest` takes the fields that are left;
  no name follows it ``.
- A pattern on an optional parameter (`{ x }: Point?`): `` a pattern
  needs a value; `Point?` may be nil ``.
- A parameter pattern that binds a name the signature already binds:
  the ordinary duplicate-binding error.

### The language server

- Hover on a bound name reads the field's type and the field's doc
  comment, the same as a hover inside the body.
- Completion inside the braces offers the fields of the annotation that
  the pattern does not yet name, with their types.
- Go to definition on a bound name lands on the field in the struct or
  the record type.
- Rename of a field offers to rename the pattern entry.
- The signature help of a call shows the parameter as its type, not as
  the pattern, because the caller passes one value.

## Drawbacks

- A signature grows wide. A pattern of six fields with types is longer
  than one name, and the reader of the call site gains nothing from it.
- Two spellings for one type: `{ bar: string }` in the pattern and
  `: Options` after it. The rule that they may not mix keeps this to
  one decision per signature.
- The emit takes a name the author did not write (`_p1`). A stack trace
  and a debugger show it.

## Alternatives

- Only the annotated form, `{ x, y }: Point`. It is the whole feature
  for a named type, and it makes the one-off options table wait for a
  `type` declaration. The inline form is what the request asked for.
- A destructuring `local` on the body's first line, which is today.
  It works, and it puts the names one line away from the signature.
- `function draw(p: Point) with { x, y }`, a separate clause. One more
  keyword for what the pattern says in place.

## Prior Art

- JavaScript: `function draw({ x, y })`, with `...rest` and defaults.
  The types come from TypeScript, written after the pattern, which is
  the annotated form here.
- Rust: `fn draw(Point { x, y }: Point)`, a pattern and its type, which
  reads as the annotated form.
- Swift: no parameter destructuring; a tuple parameter is the closest.
- Alloy: `local { name, hp = health } = player` and `for _, { x, y } in
  points`, the patterns this reuses.

## Unresolved questions

- A default for a field. `=` already renames in a pattern, so
  `{ bar = "x" }` cannot mean a default. A struct's own field default
  covers the common case; if a record type needs one, the spelling is
  open.
- Whether the whole value may also take a name, `{ x, y } as p: Point`,
  for a body that needs the table as well as its fields. `as` on a
  match scrutinee sets a precedent.
- Whether an array pattern, `[first, ...rest]: number[]`, is worth the
  same treatment. The request named the table form.
