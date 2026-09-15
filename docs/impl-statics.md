# Statics in an `impl`

## Summary

A function in an `impl` that takes no `self` is a static. It lives on
the type's table and is called with a dot: `Wallet.new()`,
`Vector3.zero()`, `Status.describe(s)`. This proposal writes down the
rules the compiler already follows for statics, and settles the open
questions the last probe rounds raised: how a static reads on a unit
enum, on a generic type, under `private`, through a namespace path, and
what the editor answers for it. No new syntax.

## Motivation

The rules for statics grew one fix at a time. A reader who knows Luau
expects `Wallet.new()` and `wallet:balance()`; Alloy gives both, but the
edge cases were decided in the compiler and not in one place:

- A unit enum is a string at runtime. `s:describe()` on a `Status` value
  cannot work, and until round 33 the checker said `Type 'Status' does
  not have key 'describe'` while the ship crashed with `attempt to call
  missing method 'describe' of string`.
- A static in `impl Box<T>` carried the impl's `<T>` even when it named
  no `T`, so `Box.of<U>(v: U)` became `Box.of<T, U>` and a caller could
  not instantiate it.
- A `private` static called from outside drew only the lint, not the
  type error a private method draws.
- A method of a struct inside a namespace, `M.Gadget.make()`, sat on a
  path the constructor check also reads, and one probe saw the
  constructor report fire on it.

The code that needs statics exists in every project: constructors,
parsers, `zero()` and `default()` values, and the methods of a unit
enum. One rule set, written down, is what a reader and the language
server can hold.

## Design

### What a static is

A function in an `impl` whose first parameter is not `self`.

```
struct Wallet as
    coins: number = 0
end

impl Wallet as
    function new(coins: number): Wallet      -- a static: no self
        return new Wallet { coins = coins }
    end

    function empty(): Wallet                 -- a static
        return Wallet.new(0)
    end

    function balance(self): number           -- a method
        return self.coins
    end
end

local w = Wallet.new(5)
local b = w:balance()
```

The emit puts both on the class table: `function Wallet.new(coins)` and
`function Wallet.balance(self)`. A static is called with `.` on the
type; a method with `:` on a value. That is the Luau rule, and Alloy
keeps it.

### The wrong call

`Wallet:new(5)` passes the table as the first argument, and every
value shifts by one. The `static_call` lint (correctness, warn)
reports it and `alloy flux --fix` writes the dot. `w.balance()` on a
method is Luau's own report: the checker says the method takes `self`.

### Unit enums

A unit enum is a string union at runtime, so a value carries no
metatable and no method. Every function in `impl Status` is a static
of the enum:

```
enum Status as
    Ready
    Pending
end

impl Status as
    function describe(self): string
        return if self == Status.Ready then "ready" else "pending"
    end
end

print(Status.describe(s))   -- the form that works
```

A colon call on a receiver the compiler knows to be a unit enum
reports at the call:

```
error(3.4): EnumError: `Status` is a unit enum, a string at runtime;
call `Status.describe(s)`
```

The recommended answer for a MIXED enum, one with a unit variant and a
payload variant, is the same rule: the enum's methods are called on the
enum. A colon call on a receiver typed as such an enum reports the same
way, because the unit variant is still a string and the call crashes on
it at runtime. A colon call stays valid only for an enum whose every
variant carries a payload. The runtime shape of enums does not change:
a table per payload variant, a string per unit variant. `self` in the
method body is the value either way.

### Generic types

A static in a generic impl carries the impl's parameter list only when
it names one of the parameters or takes `self`:

```
struct Box<T> as
    value: T
end

impl Box<T> as
    function of<U>(v: U): Box<U>           -- emits Box.of<U>
        return new Box<<U>> { value = v }
    end

    function unit(): Box<number>           -- emits Box.unit, no list
        return new Box { value = 0 }
    end

    function map<U>(self, f: (T) -> U): Box<U>   -- emits Box.map<T, U>
        return new Box { value = f(self.value) }
    end
end

local d = Box.of<<number>>(9)
```

A method's own list goes after the impl's: `map<T, U>`. A method whose
own list repeats the impl's parameter is Luau's `Duplicate type
parameter`, and stays a report. The turbofish `<<T>>` on a call leaves
the ship artifact and stays in the check artifact, where Luau reads it
as an explicit instantiation.

### `private`

A `private` static is private to the impl like a private method: a call
from outside is a type error in the editor and under `alloy flux`, and
the `private_access` lint names it in the same file. The check artifact
keeps private members out of the type's public view.

### Constructors

`new Name { ... }` is the raw construction, and a static named `new` is
the convention for a constructor that computes fields. The constructor
check, `ConstructorError`, fires when the callee IS a struct path
called like a function: `Wallet(5)`, `M.Gadget(...)`. It does not fire
for a member call on that path: `M.Gadget.make()` is a static call, and
`M.Gadget.new(5)` too. The recommended answer to the probe's report is
that rule, stated: a path whose last segment is a struct is a
constructor error when called; a path one segment longer is a member.

### Namespaces and paths

A static of a struct inside a namespace is reached through the path the
source writes: `Geo.Vec.zero()`, `M.Gadget.make()` through
`import { M }` or `import * as M`. The emit joins the path with `_`
(`Geo_Vec.zero`); every report, hover, and completion prints the path
the source wrote.

### Traits

A trait declares instance methods only. A signature without `self` is
not a trait member, because `Self` is not a type in a trait signature
and a static factory has no receiver to dispatch on. A struct that needs
a `zero()` writes it in its own impl.

### Foreign types

`impl Vector3` takes statics as it takes methods: `Vector3.zero()` in
the example above. The editor injects the static into the definitions
so the checker types it and completion lists it after `Vector3.`; the
runtime dispatcher installs it project wide.

### Constants

There is no `static` keyword and no static field. An `impl` holds
functions. A constant that belongs with a type is a `const` beside the
struct, exported with it, or a member of a namespace:

```
export const MAX_COINS = 1000

export struct Wallet as
    coins: number = 0
end
```

A reader who wants `Wallet.MAX` writes a static `function max(): number`
or the constant above. This keeps `impl` one shape: functions, on the
table, called with `.` or `:`.

### What the editor answers

- Completion after `Type.` lists the statics and the constructor;
  after `value:` the methods. A unit enum's statics list after
  `Status.` beside the variants.
- A hover on a static reads `function Wallet.new(coins: number): Wallet`;
  on a method `function Wallet:balance(self: Wallet): number`. The owner
  is the type's path as written, never the namespace alone.
- Definition, references, and rename cover a static from the
  declaration and from every dot call, within the file, through an
  import, and through a dependency project; the declaration name
  carries a `function` token with the `declaration` modifier, a method
  `method`.
- Signature help on `Wallet.new(` reads the declared parameters, with
  the impl's parameters bound from the receiver on a method.

## Drawbacks

Mixed enums lose the colon call that works today for their payload
values. The gain is one rule with no runtime crash behind it; a project
that wrote `s:area()` on a mixed `Shape` gets a report that names the
dot form, and `alloy flux --fix` can write it.

Constants stay outside the type. A reader from a class language looks
for `static MAX`; the answer is a `const` beside the struct.

## Alternatives

- A `static` keyword before the function. It adds a word for what the
  absence of `self` already says, and Luau readers know the rule.
- Give unit enums a metatable so the colon works. That changes the
  runtime shape the docs promise, costs a table per value where a
  string was, and breaks `==` against the string form the engine and
  Rojo data files use.
- Static fields on the class table. They would type as fields of every
  value in the checker's view, and a reader could not tell a static
  from an instance field at the use site.
- Do nothing. The rules keep living in the compiler, and the next probe
  round finds the next inconsistency.

## Prior Art

- Rust: associated functions without `self` are called on the type,
  `Wallet::new`; methods with `self` on the value. Alloy's `.` and `:`
  map one to one. Rust also has associated constants; Alloy declines
  them for the reasons above.
- Luau and Lua: the `.` and `:` split is the language's own; the
  `static_call` lint only names the mistake the syntax allows.
- Kotlin's companion objects and Swift's `static func` add a word.
  Alloy reads the signature instead, which is what the reader sees at
  the call.
- TypeScript enums are numbers or strings with no methods; Alloy's unit
  enum matches that runtime and puts the methods on the enum table,
  which is what a TypeScript namespace merged with the enum does.
