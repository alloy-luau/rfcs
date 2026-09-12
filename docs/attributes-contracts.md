# Attribute contracts

## Summary

An `attribute` declaration may state what the thing it sits on must
carry: a member of a given name, kind, type, and visibility. The
compiler checks it where the attribute is used. The required set may
come from the attribute's own arguments, so one attribute serves many
shapes.

## Motivation

A framework wants a type to carry certain members. Today the language
has two ways to ask, and neither fits.

A `trait` asks for methods, and `impl Trait for Name` answers. That
works when the contract is public behavior. It does not work here:

- A lifecycle method is not public API. A framework calls `Init` and
  `Start`; nothing else should. A trait member is public, so a trait
  makes the framework's own hooks part of the type's surface.
- A trait asks for the same thing every time. A framework wants to ask
  for what the user opted into: `lifecycles = [ Init, Start ]` asks for
  two methods, `lifecycles = [ Start ]` asks for one.
- A trait asks for methods. A framework may want a field.

An `interface` asks for a shape, and says so: "a field takes no private
or public: an interface is a shape other code sees whole". So it cannot
ask for a private member either.

The code a user writes today, with nothing checking it:

```alloy
@provider({ lifecycles = [ Lifecycle.Init, Lifecycle.Start ] })
impl PlayerDataProvider as
    private function Start(self) end
    -- `Init` is missing. The framework fails at run time, in a file
    -- that does not name this one.
end
```

The attribute already says what the type promises. Nothing holds the
type to it.

## Design

### The clause

An `attribute` declaration gains a body of `requires` clauses:

```alloy
attribute service on impl as
    requires public function Start(self)
    requires private field state: number
end
```

A clause reads:

    requires <visibility>? <kind> <name> <shape>

- `visibility` is `public`, `private`, or absent. Absent accepts either,
  and is the right default for a contract that does not care.
- `kind` is `function` or `field`.
- `shape` is a parameter list for a function, or `: T` for a field.

The clause is checked where the attribute is used, against the thing it
sits on.

### The required set from an argument

A clause may expand over a list parameter. The parameter's entries name
the members:

```alloy
enum Lifecycle as
    Init,
    Start,
end

attribute provider(lifecycles: Lifecycle[]) on impl as
    requires private function each lifecycles (self)
end
```

`each <param>` writes one clause per entry of that argument, with the
entry as the member name. So:

```alloy
@provider({ lifecycles = [ Lifecycle.Init, Lifecycle.Start ] })
impl PlayerDataProvider as
    private function Init(self) end
    private function Start(self) end
end
```

asks for `Init` and `Start`, and

```alloy
@provider({ lifecycles = [ Lifecycle.Start ] })
```

asks for `Start` alone.

An enum variant names the member by its own name. A string entry names
it by its text, so `[ "Init" ]` and `[ Lifecycle.Init ]` ask for the
same member. An enum reads better and completes, so the enum form is
the one the docs teach.

### No evaluation is needed

An attribute argument is already a literal the compiler reads: that is
what makes `Attributes.get` work and what `@ratelimit(5)` rests on. So
`each lifecycles` reads a constant. It does not run user code, and it
needs no type-level interpreter and no new type function.

This is the whole reason the feature is small. A proposal that computed
the required set from an expression would need one.

### Where it applies

Every target that carries members: `struct`, `impl`, `namespace`,
`enum`, `interface`, `trait`. The target list on the declaration governs
as it does today, so `on impl` means the attribute sits on an impl and
the clause reads that impl's members.

A `requires` clause on a target with no members is an error at the
declaration, not at the use.

### What the compiler emits

Nothing. A contract is a check. The attribute keeps the emit it has
today, and `Attributes.get` reads it at run time unchanged. The line
count holds because no line changes.

### The errors

A missing member:

```
error(3.x): AttributeContract: `@provider` requires a private function
`Init(self)`; `PlayerDataProvider` declares none
```

A member with the wrong visibility:

```
error(3.x): AttributeContract: `@provider` requires `Init` to be
private; `PlayerDataProvider` declares it public
```

A member with the wrong shape:

```
error(3.x): AttributeContract: `@provider` requires `Start(self)`;
`PlayerDataProvider` declares `Start(self, dt: number)`
```

Each lands on the attribute, because the attribute is the promise. A
second note points at the impl's `end`, where the member would go.

A clause the declaration cannot read:

```
error(3.x): AttributeContract: `each lifecycles` needs a list parameter;
`lifecycles` is a `string`
```

### The language server

- Hovering the attribute lists what it requires, one line per clause,
  with the argument already expanded.
- A missing member draws a code action, "write the members `@provider`
  requires", which inserts each one with its visibility, name, and
  parameter list, in the order the clauses read.
- Completion inside the target offers the missing member names first,
  with the detail `required by @provider`.
- The report lands on the attribute, so the squiggle sits on the line
  the user wrote the promise on.

## Drawbacks

`requires` becomes a word inside an attribute body, and `each` becomes
one after it. Neither is reserved elsewhere, so no existing file stops
parsing.

An attribute declaration grows a body, so `attribute name(params) on
target` gains an `as ... end` form. A declaration with no clauses keeps
the short form and reads as it does today.

The check is structural, not nominal. A type that happens to declare
`Init` satisfies the clause without meaning to. A trait says what it
means; an attribute contract says what it needs. That is the trade, and
it is the same one `interface` already makes.

A contract is checked where the attribute is used. A framework that
changes its clauses breaks every user at once, with no deprecation path
beyond the usual one.

## Alternatives

**Extend `trait` with visibility.** A private trait member would let
`impl Lifecycle for X` carry the hooks. It does not solve the second
problem: a trait cannot be asked for conditionally, so `lifecycles`
would still need a trait per combination.

**Extend `satisfies`.** `satisfies` checks a literal against a type
under contextual typing. Types carry no visibility, so it cannot ask for
a private member. Widening types to carry visibility is a much larger
change than this one, and it would change what every existing type
means.

**A type function that builds the shape.** Luau type functions can map a
value to a type. This would work, and it is the design the feature
seemed to need until the point above: the argument is a literal, so the
mapping is a read, not a computation. Reaching for a type function here
would add an evaluation order, a failure mode, and a second place a
contract can be written.

**Leave it to the framework at run time.** This is the status quo. The
failure lands in a file that does not name the one at fault, and it
lands when the game runs rather than when the project builds.

## Unresolved questions

- Should a clause be able to require a member's *type* as well as its
  signature, for a field of a generic struct?
- Should `each` accept a parameter that is a list of records, so a
  clause can name both the member and its signature from data?
- Does a contract on a `struct` read the members of its `impl` blocks,
  or only the struct's own fields? Reading both is more useful and less
  obvious.
