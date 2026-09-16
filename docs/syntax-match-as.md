# `as` on a match scrutinee

**Status**: Accepted

## Summary

`match expr as name with` binds the value under match to `name` for
every arm, guard, and `default`. The value is computed once. The name
is a plain local of the match; nothing else changes.

## Motivation

A match often needs the whole value inside an arm, next to the parts
the pattern pulls out:

```alloy
match player:FindFirstChild("Backpack") with
    case nil then
        warn("no backpack")
    case Backpack { tools } then
        -- The arm wants the backpack itself, to parent a tool.
        -- It has `tools`, not the value the match read.
end
```

Today the author writes a local first:

```alloy
local backpack = player:FindFirstChild("Backpack")
match backpack with
    ...
end
```

That is one more line, one more name at function scope, and in the
expression form of `match` there is no place for it at all:

```alloy
local label = match state:get() with
    case Loading then "..."
    case Ready { items } then `{#items} items`
    -- No way to reach the whole state here without a local above.
end
```

Rust writes `x @ Pattern` in the arm, and Swift writes `case let x`.
Alloy's `if local Pat = e` already puts the binding at the head. The
head of a `match` is the same place.

## Design

### The form

```alloy
match expr as name with
    case Pat then ...
    default ...
end
```

`as name` follows the scrutinee and comes before `with`. `name` is any
identifier. It is in scope in every `case` body, every guard (`case
Pat and cond then`), and `default`. It is not in scope after `end`.

A match of several values names each one on its own:

```alloy
match a as left, b as right with
    case Some(x), Some(y) then left, right
end
```

An `as` on one value and not another is fine.

### The name and the pattern

A pattern name that equals the alias is an error, since the two would
shadow each other on the same line:

```
error(6.x): SyntaxError: `state` is the alias of the match; a pattern
cannot bind it again
```

A bare-name pattern (`case x then`) still binds the whole value, as it
does today. With an alias that arm has two names for one value, which
is allowed and reads as the author wrote it.

### Emit

The compiler already reads the scrutinee into a local when the
expression is not a plain name, so the arms test one value:

```luau
local __m = player:FindFirstChild("Backpack")
if __m == nil then ... elseif ... end
```

With `as name`, that local takes the alias:

```luau
local backpack = player:FindFirstChild("Backpack")
if backpack == nil then ... elseif ... end
```

A scrutinee that is already a plain name emits `local name = x`. The
emit stays on the source's line: `match e as n with` is one line and
so is `local n = e`. The expression form wraps in an IIFE today and
takes the same local at the top of it.

### `if local` and `while local`

`if local Pat = e` binds the parts. For the whole value the author
already has `if local x = e where Pat = x`. No change there; the
`as` form belongs to `match`, where the head has no binding position.

### The type

`name` has the type of the scrutinee, narrowed per arm the same way a
plain-name scrutinee narrows today: in `case Ready { items }`, an alias
of `state: State` reads as the `Ready` variant.

### Errors

- `match e as with`: `expected a name after as`.
- `as` before a multi-value scrutinee's comma with no name: the same.
- An alias that shadows a name the arm's pattern binds: the error
  above, on the pattern name.
- An alias nothing reads: the ordinary `unused_variable` lint, with the
  fix that drops ` as name`.

### The language server

- Completion after the scrutinee offers `as` beside `with`.
- Hover on the alias reads the scrutinee's type, narrowed in the arm.
- Rename and references treat the alias as a local of the match.
- The unused fix removes ` as name`.

## Drawbacks

- One more meaning for `as`: import alias, struct body opener, type
  cast, and now a match alias. The position (between the scrutinee and
  `with`) makes it unambiguous to the parser and, after one reading,
  to a person.
- A reader who knows `x @ Pat` from Rust looks for the alias in the
  arm. The head position is the one Alloy already uses for `if local`.

## Alternatives

- `case x @ Pat then`, per arm, as Rust does. It aliases one arm's value
  and repeats the name across arms; the common case wants one name.
- `match local name = e with`. Longer, and `local` in a head already
  means "bind the parts" in `if local`.
- Do nothing: a local above the match, which the expression form cannot
  write.

## Prior Art

- Rust: `x @ Pattern` in an arm.
- Swift: `case let x` and `if case let`.
- OCaml: `match e with ... as x` on a pattern.
- Alloy: `if local Pat = e where cond`, the head-position binding this
  proposal follows.

## Unresolved questions

- Whether `as` may also follow a pattern (`case Ready { items } as r
  then`) for an alias narrowed to that arm's type only. The head form
  already narrows, so this would be a second spelling of the same
  thing; leave it out unless a case appears that the head form cannot
  write.
