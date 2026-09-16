# Statics: the `static` keyword for members of a type

**Status**: Accepted

## Summary

A type gains members that belong to it and not to a value, and one
word marks every one of them: `static`. A struct body takes `static`
fields and `static const` constants, read as `Wallet.made` and
`Wallet.MAX`; an `impl` takes `static function`, called as
`Wallet.new()`, `Vector3.zero()`, `Status.describe(s)`. A function in
an `impl` with no `self` and no `static` is an error with a fix, and
`self` inside a `static function` is an error. This proposal gives the
syntax, the emit, the checks, and the editor's answers, and settles
the questions the last probe rounds raised for static functions.

## Motivation

A type often owns a value that no instance owns: a counter of the
instances made, a registry, a cap, a default. Today that value lives
beside the type as a `const` or a `local`, or in a namespace, and the
reader has to know the convention to find it:

```
local wallets_made = 0             -- belongs to Wallet, lives nowhere
export const MAX_COINS = 1000      -- same

export struct Wallet as
    coins: number = 0
end

impl Wallet as
    function new(coins: number): Wallet
        wallets_made += 1
        return new Wallet { coins = math.min(coins, MAX_COINS) }
    end
end
```

A second file that imports `Wallet` cannot reach `wallets_made` at
all, and reaches `MAX_COINS` only through a second import. In a class
language the reader writes `Wallet.made` and `Wallet.MAX`, and the
type is the one place to look.

Static functions already exist, and today the absence of `self` is
the only mark. A reader scanning an `impl` has to read every
parameter list to know which functions belong to the type; a
`function new(coins: number)` and a `function balance(self)` look the
same at a glance, and a `self` typed by mistake turns a static into a
method with no report. The rules for statics also grew one fix at a
time through the probe rounds (unit enums, generic impls, `private`,
namespace paths). One word on every static member, data or function,
lets the reader and the compiler know at the declaration.

## Design

### What a user writes

```
export struct Wallet as
    coins: number = 0                 -- an instance field
    static made: number = 0           -- a static field, on the type
    static const MAX: number = 1000   -- a static constant
    private static registry: { Wallet } = {}
end

impl Wallet as
    static function new(coins: number): Wallet   -- a static function
        Wallet.made += 1
        local w = new Wallet { coins = math.min(coins, Wallet.MAX) }
        table.insert(Wallet.registry, w)
        return w
    end

    function balance(self): number               -- a method
        return self.coins
    end
end

print(Wallet.made, Wallet.MAX)
```

- `static name: T = value` in a struct body declares a static field.
  The type annotation and the initial value are both required: the
  value exists from the moment the type does, and there is no `self`
  to fill it later. `static` is contextual: it is a keyword only at
  the start of a member line in a struct body or an impl body, so
  `local static = 1` and a field named `static` stay valid Luau.
- `static const NAME: T = value` declares a static constant. This is
  new syntax: today `const` is a binding statement and nothing else
  (the upstream `const` RFC is binding-only, and Alloy matches it),
  so a `const` inside a struct body is a syntax error. The word
  `static` is what opens the body to it: the parser reads `static` at
  the start of a field line, then `const` as part of that member. The
  type may be left off when the value is a literal, as for a file
  `const`. A write to it is a `ConstError`. `const` alone in a struct
  body stays an error, and the report names the form: `` a constant
  of the type is written `static const MAX = 1000` ``.
- `private static` and `private static const` are private to the
  struct's impls, as a private field is.
- `static function name(...)` in an `impl` declares a static
  function. It takes no `self`. A function in an `impl` whose first
  parameter is not `self` and that carries no `static` is an error,
  and `alloy flux --fix` writes the word; `self` inside a `static
  function`'s parameter list or body is an error that names the
  method form. `private static function` is private to the impls.
  `async static function` is the async form. The constructor
  convention `new` is a static like any other and takes the word.
- `static` on a field goes in a `struct` body only. An `enum`, an
  `interface`, and a `type` take no `static` field: an enum's
  constants are its variants, and the rest declare shapes, not
  values. An `impl` on an enum or a foreign type takes `static
  function` as an `impl` on a struct does.
- A `trait` declares methods only; `static function` in a trait body
  is an error, because `Self` is not a type there and a static has no
  receiver to dispatch on.

### What it means

A static member lives on the type's table, `Wallet.made`, and never on
a value. `self.made` inside a method is an error that names the fix:

```
error(3.6): StructError: `made` is a static of `Wallet`; write
`Wallet.made`
```

`new Wallet { made = 1 }` is a `ConstructorError` for the same reason:
a static is not a field of a value. `@derive(Debug)`, `@derive(Eq)`,
and `Serialize` read instance fields only; a static is not part of a
value's shape.

A generic struct's static is one slot for every instantiation:
`Box.made` counts every `Box<T>`. A static may not name a type
parameter, because there is no `T` on the type's table; `static last:
T` reports `` a static has one slot for every `T`; give it a type
that names none ``.

Inside a namespace, a static is reached by the path the source writes:
`Geo.Vec.origin`. Through `import { Wallet }` or `import * as M`, the
static follows the type: `Wallet.MAX`, `M.Wallet.MAX`. An `export
struct` exports its statics with it; a private static stays private
across files as a private field does.

### The emit

The struct body's lines stay the struct's lines. Today a struct emits
its class table on the `struct` line and blanks the field lines; a
static's line writes the slot instead, so the emit stays on the
source's own lines:

```
local Wallet = {} Wallet.__index = Wallet ...   -- the struct line
                                                 -- coins: blank
Wallet.made = 0                                  -- static made
Wallet.MAX = 1000                                -- static const MAX
Wallet.registry = {}                             -- private static
```

The check artifact types the class table with the statics and keeps
them out of the value type:

```
type Wallet = { coins: number, balance: (self: Wallet) -> number }
local Wallet: { made: number, read MAX: number, new: ..., ... } = ...
```

A `static const` is a `read` member, so a write reports in the
checker as well as in the compiler. A private static goes in the
private view (`Wallet__private`) the way a private field does, and
the `private_access` lint names an outside read or write.

### Static functions

A `static function` in an `impl` is a static of the type; a
`function` with `self` is a method. The compiler reads the word, not
the parameter list, and reports the two ways they can disagree:

```
error(3.6): StructError: `new` takes no `self`; a function of the type
is written `static function new`
error(3.6): StructError: `static function balance` takes `self`; a
method is written `function balance(self)`
```

The first report carries the quickfix `Add \`static\``, and
`alloy flux --fix` applies it project wide, which is the migration for
every impl written before this proposal. The rules the compiler
follows for a static function:

- It is called with `.` on the type. `Wallet:new(5)` draws the
  `static_call` lint (correctness, warn) and `alloy flux --fix` writes
  the dot.
- A unit enum is a string union at runtime, so every function in its
  impl is a static and is written `static function describe(s:
  Status)`: a `function describe(self)` on a unit enum reports `` a
  unit enum is a string at runtime and carries no method; write
  `static function describe(s: Status)` ``. The recommended answer for
  a mixed enum, one with a unit variant beside a payload variant, is
  the same rule: the unit value is still a string and a method call
  on it crashes, so its impl takes statics only. A method with `self`
  stays valid on an enum whose every variant carries a payload.
- In `impl Box<T>`, a static carries the impl's parameter list only
  when it names a parameter: `static function of<U>(v: U): Box<U>`
  emits `Box.of<U>`; a method always carries it, `function map<U>(self,
  f)` emits `Box.map<T, U>`. The turbofish `<<T>>` on a call leaves the
  ship artifact and stays in the check artifact.
- A `private static function` is a type error from outside and the
  `private_access` lint in the same file, as a private method is.
- `ConstructorError` fires when the callee IS a struct path called
  like a function, `Wallet(5)` or `M.Gadget(...)`, never for a member
  of that path: `M.Gadget.make()` is a static call.
- `impl Vector3` takes `static function zero(): Vector3` as it takes
  methods; the editor injects it into the definitions and the runtime
  dispatcher installs it.

### What the editor answers

- Completion after `Wallet.` lists the static fields and constants
  (kind `field` and `constant`) beside the static functions and the
  constructor; after `w.` and `w:` the instance members only.
- A hover on `made` reads `static made: number` with `A static of
  \`struct Wallet\`.`; on `MAX` `static const MAX: number = 1000`; on
  `new` `static function Wallet.new(coins: number): Wallet`; on
  `balance` `function Wallet:balance(self: Wallet): number`.
- Definition, references, and rename cover a static from the struct
  body and from every `Wallet.made` site, within the file, through an
  import, and through a dependency project. `Wallet.made` and `w.coins`
  never share a rename.
- A `self.made` report carries a quickfix `Rewrite as \`Wallet.made\``.
- The formatter keeps `static`, `private static`, and `async static`
  in front of the field or the function, in that order, one form,
  stable on a second pass. `alloy fmt` does not add the word; `alloy
  flux --fix` does.

### Errors, in the compiler's words

- `static made: number` with no value: `` a static needs a value: `static
  made: number = 0` ``.
- `const MAX = 1000` in a struct body: `` a constant of the type is
  written `static const MAX = 1000` `` (today a syntax error with no
  guidance).
- A no-`self` function without the word: `` `new` takes no `self`; a
  function of the type is written `static function new` `` (quickfix
  `Add \`static\``).
- `self` in a `static function`: `` `static function balance` takes
  `self`; a method is written `function balance(self)` ``.
- `static function` in a trait: `` a trait declares methods; a static
  has no receiver ``.
- `static` in an enum, interface, trait, or type: `` `static` belongs in
  a `struct` body ``.
- `self.made`: `` `made` is a static of `Wallet`; write `Wallet.made` ``.
- `new Wallet { made = 1 }`: `` `made` is a static of `Wallet`, not a
  field of a value ``.
- `Wallet.MAX = 2`: `` `MAX` is a `static const`; its value is set once ``.
- `static last: T`: `` a static has one slot for every `T`; give it a
  type that names none ``.

## Drawbacks

`static` becomes a reserved word in a struct body. Luau does not
reserve it, so `local static = 1` stays valid and a field named
`static` needs no change: the word is contextual, read only at the
start of a field line, the way `read` and `write` are.

A static field is shared state on a type. The proposal makes that
visible in the syntax and reachable across files, which is the point,
and also a foot-gun for code that treated a file `local` as private by
accident. `private static` is the answer for that code.

Every impl written so far has static functions without the word, so
every project gets a report per static on the day this lands. The
report is one quickfix, `alloy flux --fix` applies it to a whole
project in one run, and the emit does not change, so the migration is
a diff of one word per static and nothing at runtime. The upstream
watch matters here: Luau's class proposals (240, 242, 247) spell a
class static with the same word, so an Alloy impl reads the way an
upstream class will.

A project that wrote `Wallet.made = 0` after the struct by hand keeps
working: the emit is the same assignment. It gains a type and a hover
by moving the line into the body.

## Alternatives

- Keep the convention: a `const` or `local` beside the struct, or a
  namespace. It works today and stays valid; it does not travel with
  the type across an import, and the reader has to know it.
- Keep static functions unmarked, as today, and put `static` on
  fields only. It saves one word per static and the migration diff;
  it leaves the reader reading parameter lists, and a `self` typed by
  mistake still turns a static into a method with no report. The
  user's own read, on the day of the decision: the word belongs in
  front of the function too, and a function with no `self` and no
  word is an error.
- Exempt `new` from the word, since every struct has one. One
  exception to a one-word rule is the kind of thing a reader has to
  learn; `static function new` reads fine and the fix writes it.
- A `class` keyword with `static` inside it. Upstream Luau's class RFCs
  (240, 242, 247) may bring one; Alloy's `struct` plus `impl` is the
  shape now, and a static field on a struct maps onto a class static
  if a class lands.
- Statics in an `impl` body (`impl Wallet as static made = 0 end`).
  The impl is where functions live and where a second file may add
  them; a slot with a value belongs with the declaration, in one place.
- Do nothing. Static functions stay as they are and data stays beside
  the type; the probe rounds keep finding the same questions.

## Prior Art

- Rust: associated constants (`const MAX: u32`) on a type, no static
  fields; associated functions without `self` called on the type.
  Alloy takes the constant and adds the field, since Luau has a live
  table to hold it and no ownership rules to forbid it.
- Kotlin companion objects, Swift `static var`, `static let`, and
  `static func`: a keyword on the member, read as `Type.member`.
  Alloy's `static` reads the same way on a field and on a function.
- TypeScript, Java, C#: `static` on a class field or method, called
  on the type. This proposal reads the same on both, and adds the
  report that TypeScript lacks: a function with no `this` is still a
  method there, and a caller finds out at the call.
- Lua and Luau: a "static" is a plain table member, `Wallet.made = 0`
  after the table. The emit is exactly that; the proposal gives it a
  place in the declaration, a type, and a name the editor knows.
