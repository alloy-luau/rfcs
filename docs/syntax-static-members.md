# Statics: the `static` keyword for members of a type

## Summary

A struct gains members that belong to the type and not to a value:
`static` fields and `static const` constants, written in the struct
body and read as `Wallet.count` or `Wallet.MAX`. A function in an
`impl` that takes no `self` stays a static without any keyword, as it
is today: `Wallet.new()`, `Vector3.zero()`, `Status.describe(s)`. The
keyword is for data, where the absence of `self` cannot say it. This
proposal gives the syntax, the emit, the checks, and the editor's
answers, and settles the questions the last probe rounds raised for
static functions.

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

Static functions already exist and need no word: `function new()` with
no `self` is one. The rules for them grew one fix at a time through
the probe rounds (unit enums, generic impls, `private`, namespace
paths); this proposal writes them down beside the new keyword so the
type's statics are one family.

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
    function new(coins: number): Wallet          -- a static function
        Wallet.made += 1
        local w = new Wallet { coins = math.min(coins, Wallet.MAX) }
        table.insert(Wallet.registry, w)
        return w
    end

    function balance(self): number
        return self.coins
    end
end

print(Wallet.made, Wallet.MAX)
```

- `static name: T = value` in a struct body declares a static field.
  The type annotation and the initial value are both required: the
  value exists from the moment the type does, and there is no `self`
  to fill it later.
- `static const NAME: T = value` declares a static constant. The type
  may be left off when the value is a literal, as for a file `const`.
  A write to it is a `ConstError`.
- `private static` and `private static const` are private to the
  struct's impls, as a private field is.
- `static` goes only on a field of a `struct`. It is not written on a
  function: a function in an `impl` with no `self` is a static already,
  and a `static function` reports `` `static` goes on a field; a
  function with no `self` is a static ``. An `enum`, an `interface`, a
  `trait`, and a `type` take no `static`: an enum's constants are its
  variants, and the rest declare shapes, not values.

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

### Static functions, restated

A function in an `impl` whose first parameter is not `self` is a
static of the type. The rules the compiler follows:

- It is called with `.` on the type. `Wallet:new(5)` draws the
  `static_call` lint (correctness, warn) and `alloy flux --fix` writes
  the dot.
- A unit enum is a string union at runtime, so every function in its
  impl is a static: `Status.describe(s)`. A colon call on a receiver
  typed as a unit enum reports `` `Status` is a unit enum, a string at
  runtime; call `Status.describe(s)` ``. The recommended answer for a
  mixed enum, one with a unit variant beside a payload variant, is the
  same report: the unit value is still a string and the colon call
  crashes on it. A colon call stays valid only for an enum whose every
  variant carries a payload.
- In `impl Box<T>`, a static carries the impl's parameter list only
  when it names a parameter or takes `self`: `function of<U>(v: U):
  Box<U>` emits `Box.of<U>`; `function map<U>(self, f)` emits
  `Box.map<T, U>`. The turbofish `<<T>>` on a call leaves the ship
  artifact and stays in the check artifact.
- A `private` static function is a type error from outside and the
  `private_access` lint in the same file, as a private method is.
- `ConstructorError` fires when the callee IS a struct path called
  like a function, `Wallet(5)` or `M.Gadget(...)`, never for a member
  of that path: `M.Gadget.make()` is a static call.
- A trait declares instance methods only; a signature without `self`
  is not a trait member, since `Self` is not a type there.
- `impl Vector3` takes statics as it takes methods; the editor injects
  them into the definitions and the runtime dispatcher installs them.

### What the editor answers

- Completion after `Wallet.` lists the static fields and constants
  (kind `field` and `constant`) beside the static functions and the
  constructor; after `w.` and `w:` the instance members only.
- A hover on `made` reads `static made: number` with `A static of
  \`struct Wallet\`.`; on `MAX` `static const MAX: number = 1000`.
- Definition, references, and rename cover a static from the struct
  body and from every `Wallet.made` site, within the file, through an
  import, and through a dependency project. `Wallet.made` and `w.coins`
  never share a rename.
- A `self.made` report carries a quickfix `Rewrite as \`Wallet.made\``.
- The formatter keeps `static` and `private static` in front of the
  field, one form, stable on a second pass.

### Errors, in the compiler's words

- `static made: number` with no value: `` a static needs a value: `static
  made: number = 0` ``.
- `static` on a function: `` `static` goes on a field; a function with
  no `self` is a static ``.
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

A project that wrote `Wallet.made = 0` after the struct by hand keeps
working: the emit is the same assignment. It gains a type and a hover
by moving the line into the body.

## Alternatives

- Keep the convention: a `const` or `local` beside the struct, or a
  namespace. It works today and stays valid; it does not travel with
  the type across an import, and the reader has to know it.
- `static function` as a required or optional word. It adds a word for
  what the absence of `self` already says, and every static function
  written so far would gain a diff. Rejected; the report on `static
  function` names the rule.
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
- Kotlin companion objects, Swift `static var` and `static let`, C#
  and Java `static`: a keyword on the member, read as `Type.member`.
  Alloy's `static` reads the same way on a field and is absent on a
  function, where the signature already says it.
- TypeScript: `static` on a class field or method. The field form
  matches this proposal; the method form is the one Alloy declines.
- Lua and Luau: a "static" is a plain table member, `Wallet.made = 0`
  after the table. The emit is exactly that; the proposal gives it a
  place in the declaration, a type, and a name the editor knows.
