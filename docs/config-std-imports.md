# The std by import

**Status**: Accepted

## Summary

A file imports the std names it uses:
`import { HashMap, Set } from "@alloy/std/collections"`. The std is
laid out in modules by subject, and `@alloy/std` re-exports all of
them. The names the language owns stay ambient: `Future`, `Result`,
`Ok`, `Err`, `Array`, and the operator traits. A project that wants
more ambient names says so: `[std] globals = "all"` makes every std
name ambient, and a list, `globals = ["Signal", "Iter"]`, makes those
names ambient. `Serialize` and `Deserialize` move to
`@alloy/std/serde`, and `@derive` takes them as imported names. The
emit does not change in any case.

## Motivation

Every std name was ambient. `HashMap`, `Queue`, `Iter`, `Result`,
`Ok`, `Err`, `Signal` and about fifteen more were in scope in every
file, with no line saying so:

```alloy
local m = new HashMap()
local r: Result<number, string> = Ok(1)
```

That is short, and it costs three things.

A reader cannot tell where a name comes from. `HashMap` and a name the
project declares in another file look the same at the use site, and
only `Result` has a chapter a reader would think to open.

A name the author wants is taken. A game with its own `Signal` or its
own `Set` either renames it or shadows the std one, and the shadow is
silent.

The ambient list grows. Every std type added takes a name from every
project that exists, and the project finds out when a build breaks.

The runtime already does not need the ambient names. The emit
qualifies every one of them:

```alloy
local m = new HashMap()
```

```luau
local __alloy = require("@alloy") local m = __alloy.HashMap.new()
```

So the ambient name is a source convenience, not a runtime fact, and
turning it off costs nothing at run time.

## Design

### The option

```toml
[std]
# "none" (the default), "all", or a list of names to keep ambient.
globals = "none"
```

Three values:

- `"none"`, the default. A file imports each std name it uses, except
  the names the language owns.
- `"all"`. Every std name is ambient, the behavior before this
  proposal.
- A list, `["Signal", "Iter"]`. Those names are ambient too, and every
  other std name needs an import. A name that is not a std name is an
  error when the config loads, and the error lists the std names.

The list is the value to teach for a project that wants a few names
everywhere. It names what the project treats as its own vocabulary, a
`Signal` in a game built on events, an `Iter` in one built on
pipelines. It never has to name the language's own names, which the
next sections keep ambient.

The option sits in its own `[std]` table rather than in `[emit]`,
because it changes name resolution and not the emit. `alloy init`
writes `globals = "none"`, so a new project shows the key.

A file outside any project reads the default, `"none"`.

### The modules

The std has nine modules. One module holding every name is a list to
scroll, not a library to learn, so the names sit by subject:

| module | holds |
|---|---|
| `@alloy/std/collections` | `HashMap`, `Set`, `Queue`, `Heap`, `Array`, `Symbol`, `BitSet` |
| `@alloy/std/iter` | `Iter` |
| `@alloy/std/result` | `Result`, `Ok`, `Err` |
| `@alloy/std/async` | `Future`, `Scope` |
| `@alloy/std/signal` | `Signal`, `SignalConnection`, `Signalish` |
| `@alloy/std/traits` | `Display`, `Debug`, `Clone`, `Default`, `Eq`, `PartialEq`, `Ord`, `Add`, `Sub`, `Mul`, `Div` |
| `@alloy/std/serde` | `Serialize`, `Deserialize`, and the attributes `rename`, `rename_all`, `skip`, `deny_unknown_fields` |
| `@alloy/std/types` | `Partial`, `Readonly`, `Sink` |
| `@alloy/std/roblox` | `R15Character`, `R6Character`, `Attributes` |

A name sits in one module. `Sink` is a mapped type, the write half of
`Readonly`, so it sits with the type utilities and not with the
signals. The table lists the names a source can write; a type the
runtime uses on its own, such as `Awaitable`, has no entry.

`@alloy/std` re-exports every module, so a file may name the group or
name the library:

```alloy
import { HashMap, Set } from "@alloy/std/collections"
import { Signal } from "@alloy/std/signal"
import type { Partial } from "@alloy/std/types"

-- The same names, through the facade.
import { HashMap, Signal } from "@alloy/std"
```

The grouped form is the one to teach, because it says what a name is
for. The facade exists so a file with four names from four modules is
one line.

### The forms an import takes

Every import form a module takes works on the std:

- A list, `import { HashMap }`, binds each name under its own name.
- An alias, `import { HashMap as Map }`, binds the alias. `Map` then
  reads as the value and as the type, `Map<string, number>`.
- A star import, `import * as c from "@alloy/std/collections"`, binds
  the module. `c.HashMap.new()` and `c.HashMap<K, V>` both work, and
  `c.Signal.new<<number>>()` takes its type pack as the bare form does.
  The local reaches the names its module exports and no more:
  `c.Signal` reports that `Signal` is in `@alloy/std/signal`, and a
  helper of the runtime, such as `try_block`, reports as no name of the
  std. Completion after `c.` lists the same names.
- A type import, `import type { Partial }`, reads the same way.

A default import, `import std from "@alloy/std"`, is an error: the std
has no default export, and the report names the star form.

A std import is allowed under every value of the option. Under `"all"`
it adds nothing, and a file that wants to be explicit may write it.
An unused std import draws `unused_import`, as any import does.

### The names the language owns

A keyword or an operator writes std names the author never types, and
a failure idiom reads them back. Asking a file to import a name for a
keyword it wrote would be absurd, so these names are the language's,
and they stay ambient under every value of the option:

| written by | the names |
|---|---|
| `async`, `await` | `Future` |
| `try`, and a `Result` return | `Result`, `Ok`, `Err` |
| `[ ]` literals and `T[]` types | `Array` |
| `==`, `<`, `+`, `-`, `*`, `/`, `tostring`, `clone` | the traits of `@alloy/std/traits` |

The traits are in the table because they are the hooks the operators
call. `@derive(Eq, Debug, Clone, Default)` and a bound `<T: Ord>` need
no import for the same reason. A name only the author writes is importable: the
collections, the signals, the iterators, the type utilities, and
serde.

The owned names are still importable from their module, for a file
that prefers to be explicit.

### Serialize and Deserialize

`@derive(Serialize)` wrote both directions: `to_table`, `serialize`,
and `from_table`. The two halves now split, the way Rust's serde
splits them:

- `@derive(Serialize)` writes `to_table` and `serialize`. `serialize`
  is what a `T: Serialize` bound asks for.
- `@derive(Deserialize)` writes `from_table`, which builds the struct
  from the table `Serialize` writes.

A struct that round-trips derives both. A field whose type is a struct
of the file that derives the same half goes through that struct's own
function, so the table nests on the way out and the metatable comes
back on the way in. `@rename` and `@skip` apply to both halves.

Both names live in `@alloy/std/serde`, and a derive argument is a name
like any other. A file imports it the way it imports a type:

```alloy
import { Serialize, Deserialize } from "@alloy/std/serde"

@derive(Serialize, Deserialize, Eq)
struct Save
    coins: number
end

local function store<T: Serialize>(value: T) return value:serialize() end
```

So the answer to "how does a derive argument get imported" is the
answer for every other name: an import binds it, and a user trait that
the derive can write is imported from its own module the same way.

The options the serde derives read are std names of the same module:
`@rename`, `@rename_all`, `@skip`, and `@deny_unknown_fields`. Each one
needs its import, as `Serialize` does, and `alloy flux --fix` writes
it. A star import reaches the options and the derives through the
module:

```alloy
import * as serde from "@alloy/std/serde"

@derive(serde.Serialize, serde.Deserialize)
@serde.deny_unknown_fields
struct Save
    @serde.rename("hp")
    health: number
end
```

`@alias` stays built in. It gives a field a second key that reads and
writes the same slot, which a struct does with or without serde.

### What the modules cost

Nothing at run time. A std import writes no `require`: the name renders
as `__alloy.Name` wherever it stands, the way an ambient name does.

```alloy
import { HashMap } from "@alloy/std/collections"

local m = new HashMap()
```

```luau
local __alloy = require("@alloy") local m = __alloy.HashMap.new()
```

So the layout is a naming device. The compiler holds one table from
name to module, the modules resolve to no path on disk, and two files
that import one name through different specs produce identical output.
A star import writes one `require` of the runtime, the module the file
already requires.

The prefix is `@alloy/std` rather than `@alloy` so the toolchain keeps
room for a surface that is not the library, `@alloy/testing` for the
assert macros, for example.

### The report

A std name that the file does not reach reports once per name, at its
first use, with the line to write:

```
error(3.2): ImportError: `HashMap` is in the std; write `import { HashMap } from "@alloy/std/collections"`
```

The message names the import rather than saying the name is unknown,
because the name is not a typo and the reader needs the line, not the
diagnosis. The one table from name to module means the report can name
the right module, and it means a name imported from the wrong one is
its own report:

```
error(3.2): ImportError: "@alloy/std/collections" has no `Signal`; it is in "@alloy/std/signal"
```

A module the std does not have lists the ones it has. A name that is
ambient under the project's option draws no report.

### Migration

`alloy flux --fix` and `alloy lint --fix` write every missing import.
A name joins an `import { } from` list the file already has for the
facade or for its module, and the rest go on new lines below the last
import, one line per module. A project that updates the compiler runs
the command once. A project that wants the old behavior sets
`globals = "all"`.

A struct that derived `Serialize` and read its value back with
`from_table` adds `Deserialize`. The type checker reports the missing
`from_table`, so the gap is visible.

The option is per project. A library and the game that uses it may
disagree, because the setting decides how a file is written, not what
the build produces.

### The language server

- A std name in a completion list names its module in its detail,
  `alloy:std:collections`. Where the file does not reach the name,
  accepting it writes its import.
- `@derive(` offers `Serialize` and `Deserialize` with their
  `@alloy/std/serde` import, and the owned traits with none. The
  attribute list after `@` stays the list for the target below it.
- Inside `from "@alloy/std/"` the list is the modules, and inside the
  braces of a std import it is the names that module holds.
- The report takes a quick fix that writes its import, and a file with
  several missing names takes one action that writes them all.
- A type hint that names a std type the file does not reach writes the
  import with the annotation, so the accepted text compiles.
- A hover on an imported std name reads its std doc, the same as an
  ambient one.

## Drawbacks

- A breaking default. Every file written before this proposal that
  uses a library name bare stops compiling until it imports the name.
  `alloy flux --fix` is the one command that fixes a project, and
  `globals = "all"` keeps the old behavior for a project that wants it.
- More lines. A file that uses four std names gains an import line.
- A setting that changes what compiles. A file copied between two
  projects may stop compiling, and the report above is what it gets.
- The serde options need an import where they were built in, so a
  file that used `@rename` with no import reports until it adds one.
  The report carries the fix.
- The serde split breaks a struct that derived `Serialize` and read
  back with `from_table`. The type checker names the missing function.

## Alternatives

- Keep the std ambient by default and let a project opt out with
  `globals = "none"`. It breaks no existing file. It also keeps every
  cost in the Motivation for every project that does not know the
  option exists, and a library name taken by default is the problem
  this proposal exists to fix.
- `import { HashMap } from "@alloy"`, with no `/std`. It is the module
  the emit already requires, so it is one fewer idea. It leaves no room
  for a second surface and it reads as the toolchain rather than the
  library.
- One flat `@alloy/std` with no modules. It is the smallest change and
  it makes fifty names one list, which is the state the modules exist
  to fix. The facade keeps this form available for the file that wants
  it.
- Deeper paths, `@alloy/std/collections/hash_map`, one module per type.
  It matches C++ headers and it makes a file that uses four
  collections carry four import lines for one subject.
- A per-file directive, `--@alloy-no-globals`, instead of a project
  setting. It makes the choice local, and it puts a compiler flag in
  every file of a project that wants it everywhere.
- Keep one `Serialize` that writes both directions. It is less typing,
  and it hides the direction a struct supports. A payload that only
  goes out, or a table that only comes in, says so under the split.
- Import the operator traits too. It is the stricter rule, and it asks
  for an import for `==` on a struct, which a reader never connects to
  a library name.

## Prior Art

- Rust: `std::collections::HashMap` groups by subject, and a small
  prelude is ambient. This proposal is that shape: the prelude is the
  names the language owns, and a project may widen it. serde splits
  `Serialize` and `Deserialize` the same way.
- Python: no ambient library; every name is an import, and builtins are
  the fixed small set.
- Go: no ambient library, and an unused import is an error.
- Luau and Roblox: globals are ambient and a script cannot opt out,
  which is the state Alloy inherited and this proposal replaces.
