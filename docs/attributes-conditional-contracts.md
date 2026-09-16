# Conditional attribute contracts

**Status**: Noted for later

The contract feature ships without this. The user asked for it on
2026-09-15 while writing a provider attribute, and this file holds the
idea so the next design pass starts from it.

## Summary

A `requires` clause may end in `where <condition>`, a condition over
the attribute's own arguments. The clause applies only when the
condition holds. The compiler reads the condition from the literal
arguments at the use site, the same way `each` reads a list today.
Nothing runs.

## Motivation

A contract that comes from the attribute's arguments already exists:
`each lifecycles` writes one clause per entry. The next request is a
clause that depends on a flag:

```alloy
attribute provider(lifecycles: Lifecycle[], dbgVal: boolean) on impl as
    requires private function each lifecycles (self)
    requires private field test: number where dbgVal
end
```

A framework that turns on a debug mode wants a debug member only then.
Today the author writes two attributes, or one attribute that requires
the member always, and the type carries a field it does not use.

## Design

### The condition

`where` on a clause takes a condition over the attribute's parameters
and literals only, the same word and the same shape as `for x in xs
where cond` and `if local x = e where cond`:

- a boolean parameter: `where dbgVal`
- a comparison of a parameter with a literal: `where mode ==
  "server"`, `where count > 0`
- `not`, `and`, `or` over those

`where` sits last on the clause, after the member's shape. With
`each` it reads per entry: `requires private function each
lifecycles (self) where dbgVal` writes the clauses only when the flag
is on.

A parameter of list type may appear only as `#list` (the count) or in
`each`. A call, an index, a table, or any name that is not a parameter
is an error:

```
error(3.x): AttributeContract: a contract condition reads the
attribute's parameters and literals; `os.time()` is a call
```

### What the compiler does

The arguments of an attribute use are literals the compiler reads
today. So the compiler folds the condition at each use site: it
substitutes the arguments, evaluates the comparison, and keeps the
clauses whose condition holds. The required set is then a plain list,
and the existing check runs. This is a fold over constants, not an
interpreter; the grammar above is small enough that the fold is a
match over five node kinds.

A use site that gives no value for a parameter the condition reads
takes the parameter's default. A parameter with no default and no
value is already an error.

### Emit

Nothing. An attribute body emits nothing today, and the condition adds
no output. The check runs at build time and reports at the use site.

### Errors

A missing member reports as today, with the condition named:

```
error(3.x): AttributeContract: `@provider` asks for `private field
test: number` when `dbgVal` is true; `PlayerDataProvider` has no `test`
```

### The language server

The required-member completion inside an attributed `impl` reads the
folded set, so it offers `test` only when the use site turns `dbgVal`
on. The hover on `@provider` at a use site lists the clauses that hold
there.

## Drawbacks

- The condition grammar is a subset the docs must draw a line around.
  Every extension of that subset is a step toward a type-level
  language, which the contracts RFC set aside on purpose ("No
  evaluation is needed").
- Two use sites of one attribute may now ask for different members,
  which makes the attribute harder to read as a single shape.

## Alternatives

- Two attributes, `@provider` and `@provider_debug`, with the debug
  contract in the second. Works today, no new syntax. Costs one name
  per flag and puts the flag in the attribute name instead of an
  argument.
- An `if dbgVal then ... end` block inside the attribute body, with
  `else` and `elseif`. It groups several clauses under one condition,
  but `if` inside a declaration reads as code that runs, and a block
  needs `end` handling in a body that has none today. The `where`
  suffix keeps one clause per line and reuses a word the language
  already gives this meaning.
- Do nothing: the framework checks at run time, which is the failure
  the contracts RFC exists to remove.

## Prior Art

- Rust's `#[cfg(...)]` is an attribute whose condition is a fold over
  build flags, never a computation. The grammar here is the same kind:
  names, literals, `not`/`and`/`or`.
- TypeScript conditional types compute over types at check time. That
  is the interpreter this proposal avoids.

## Unresolved questions

- Whether a condition may read the `each` entry (`where lifecycle ~=
  Lifecycle.Init`), which needs the entry to have a name.
- Whether a condition may read a struct field of a record argument
  (`if options.debug then`), which needs the record parameter shape
  from the contracts RFC's open question.
