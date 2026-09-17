# The std by import

**Status**: Accepted

## Summary

The std stays ambient by default. A project that wants the opposite
asks for it: `[std] globals = "none"` in `alloy.toml` turns the ambient
names off, and a file then writes
`import { HashMap, Set } from "@alloy/std/collections"`. The std is
laid out in modules by subject, with `@alloy/std` re-exporting all of
them, so a file may import from the group or from the one name. The
option also takes a list, so a project keeps the few names it wants
everywhere and imports the rest. A project that sets nothing reads
exactly as it does today, and the emit does not change in any case.

## Motivation

Every std name is ambient today. `HashMap`, `Queue`, `Iter`, `Result`,
`Ok`, `Err`, `Signal` and about fifteen more are in scope in every
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
# "all" (the default), "none", or a list of names to keep ambient.
globals = "none"
```

Three values:

- `"all"`, the default and today's behavior. Every std name is ambient.
- `"none"`. No std name is ambient. A file imports what it uses.
- A list, `["Result", "Ok", "Err"]`. Those names stay ambient and every
  other std name needs an import.

The list is the value to teach. It names what a project treats as part
of its own vocabulary, a `Signal` in a game built on events, an `Iter`
in one built on pipelines. It does not have to name the language's own
types, which the next section keeps ambient whatever the option says.

### The modules

The std has about fifty exported names. One module holding all of them
is a list to scroll, not a library to learn, so the names sit in
modules by subject:

| module | holds |
|---|---|
| `@alloy/std/collections` | `HashMap`, `Set`, `Queue`, `Heap`, `Array`, `ReadArray`, `WriteArray`, `Container` |
| `@alloy/std/iter` | `Iter`, `Iter2`, `Iter3` |
| `@alloy/std/result` | `Result`, `Ok`, `Err`, and the result methods |
| `@alloy/std/async` | `Future`, `Awaitable`, `Settled`, `Scope` |
| `@alloy/std/signal` | `Signal`, `SignalConnection`, `Signalish`, `Sink` |
| `@alloy/std/traits` | `Add`, `Sub`, `Mul`, `Div`, `Eq`, `PartialEq`, `Ord`, `Clone`, `Debug`, `Display`, `Serialize`, `Deletable`, `Destroyable` |
| `@alloy/std/types` | `Partial`, `Readonly`, and the other type utilities |
| `@alloy/std/roblox` | `R15Character`, `R6Character`, `Remote`, `RemoteSpec`, `RemoteCalls`, `Attribute` |

A name in this table may also be one the language owns; the next
section says which, and those stay ambient whatever the option says.

`@alloy/std` re-exports every module, so a file may name the group or
name the library:

```alloy
import { HashMap, Set } from "@alloy/std/collections"
import { Signal } from "@alloy/std/signal"
import type { Result } from "@alloy/std/result"

-- The same names, through the facade.
import { HashMap, Signal } from "@alloy/std"
```

The grouped form is the one to teach, because it says what a name is
for. The facade exists so a file with four names from four modules is
one line, and so a project that moves to `globals = "none"` has one
spec to rewrite to rather than eight.

### The names the language owns

A keyword emits std types the author never writes. `remote Ping() from
client` emits a value typed `RemoteSpec`; an `async function` emits a
`Future`; `attribute tag on function` emits an `Attribute`; `try do
... end` yields a `Result`. Asking a file to import a name it never
typed, for a keyword it did, would be absurd.

So those names are the language's, not the library's. They stay ambient
under every value of the option, `"none"` included:

| keyword | the names it owns |
|---|---|
| `remote` | `Remote`, `RemoteSpec`, `RemoteCalls` |
| `async`, `await` | `Future`, `Awaitable`, `Settled` |
| `attribute` | `Attribute` |
| `try`, and a `Result` return | `Result`, `Ok`, `Err` |

The rule behind the table: a name the compiler emits for a keyword is
ambient, and a name only the author writes is importable. A project
that sets `globals = "none"` still writes `remote Ping() from client`
and still reads `Ok(v)` out of a `try`, with no import line for either.

`Ok` and `Err` are in the table because `try do ... end` produces them
and a `match` reads them. That is the failure idiom the earlier section
suggested a project would list by hand; it is not a choice, it is the
language.

These names are still importable from their module, for a file that
prefers to be explicit. The import is allowed and adds nothing, the way
an import under `"all"` does.

### What the modules cost

Nothing at run time. Every spec here lowers to the same require the
emit already writes, whichever module the name came from:

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

A type import costs nothing at run time either, and follows the
project's existing `erase_type_imports` setting.

The prefix is `@alloy/std` rather than `@alloy` so the toolchain keeps
room for a surface that is not the library, `@alloy/testing` for the
assert macros, for example.

### The report

Under `globals = "none"`, a std name with no import reads as an unknown
name, with the import to write:

```
error(3.2): UnknownName: `HashMap` is a std type; write
`import { HashMap } from "@alloy/std/collections"`
```

The message names the import rather than saying the name is unknown,
because the name is not a typo and the reader needs the line, not the
diagnosis. The one table from name to module means the report can name
the right module, and it means a name imported from the wrong one is
its own report:

```
error(3.2): UnknownName: `@alloy/std/collections` has no `Signal`; it
is in `@alloy/std/signal`
```

`alloy flux --fix` writes both, and the editor offers them as a quick
fix beside the type auto-import quickfix that already exists.

A name that is ambient under the project's list draws no report.

### Migration

`alloy flux --fix` on a project that has just set `globals = "none"`
adds every missing import, one line per file, sorted and merged with
the file's existing imports. That is the same rewrite the auto-import
quickfix performs, applied to the whole project, so switching the
option is one command rather than a hand edit per file.

The option is per project. A library and the game that uses it may
disagree, because the setting decides how a file is written, not what
the build produces.

### The language server

- Under `"none"`, completion still offers every std name, and accepting
  one inserts its import, the way a type auto-import does now.
- A hover on an imported std name reads its doc, unchanged.
- Under a list, the ambient names complete with no import and the rest
  complete with one, so the list is visible in the editor rather than
  only in the file.

## Drawbacks

- Two ways to read the same program. A reader of one project sees
  `HashMap` bare and of another sees it imported. The default keeps
  today's behavior, so the split only exists where a project asked for
  it.
- More lines. A file that uses four std names gains an import line, and
  a project that sets `"none"` gains one in most files.
- A setting that changes what compiles. A file copied between two
  projects may stop compiling, and the report above is what it gets.
- The list value needs a decision per project, and a project that picks
  badly gets churn later when it changes its mind. `alloy flux --fix`
  covers the change in both directions.

## Alternatives

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
- Drop the ambient names outright, with no option. It is the cleaner
  language and it breaks every file written so far.
- Keep only the option to shadow, so a project's own `Signal` wins over
  the std one silently. That is the behavior this proposal exists to
  make visible.

## Prior Art

- Rust: `std::collections::HashMap` groups by subject, and a small
  prelude is ambient. This proposal is that shape, with the prelude
  chosen per project rather than by the language.
- Python: no ambient library; every name is an import, and builtins are
  the fixed small set.
- Go: no ambient library, and an unused import is an error.
- Luau and Roblox: globals are ambient and a script cannot opt out,
  which is the state Alloy inherits and this proposal makes a choice.

## Unresolved questions

- Which names a project should keep ambient when it writes a list. The
  default stays `"all"`, so this is guidance for the docs rather than a
  change to the language.
- Where a name belongs when two modules could hold it. `Attribute` is
  Roblox metadata and also compiler metadata; `Sink` is a signal and
  also a collection. The table above makes a call on each, and the
  first project to disagree is the evidence to move one.
- Whether a module may be imported whole, `import * as collections
  from "@alloy/std/collections"`, which the language already spells for
  any other module.
- Whether an unused std import should draw the existing
  `unused_import` lint, which would make a copied file noisy while it
  is being adapted.
- Whether `@alloy/std` should also be importable under `"all"`, so a
  file may be explicit in a project that does not require it. Allowing
  it costs nothing and gives one name two spellings in one project.
- Whether the setting belongs in `[std]` or in the existing `[emit]`
  table, given that it changes name resolution and not the emit.
