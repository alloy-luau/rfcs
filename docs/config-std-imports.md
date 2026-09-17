# The std by import

**Status**: Accepted

## Summary

The std stays ambient by default. A project that wants the opposite
asks for it: `[std] globals = "none"` in `alloy.toml` turns the ambient
names off, and a file then writes
`import { HashMap } from "@alloy/std"`. The option also takes a list,
so a project keeps the few names it wants everywhere and imports the
rest. A project that sets nothing reads exactly as it does today, and
the emit does not change in any case.

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

The list is the value to teach. `Result`, `Ok` and `Err` are the
language's failure idiom and read in almost every file, the way `print`
does; `HashMap` and `Heap` appear in a few. A project that writes the
list says which names it treats as part of the language.

### The import

```alloy
import { HashMap, Queue } from "@alloy/std"
import type { Result } from "@alloy/std"
```

`@alloy/std` is a spec the compiler knows, not a path it resolves on
disk. It lowers to the same runtime require the emit already writes, so
an import adds no module and no load:

```luau
local __alloy = require("@alloy") local m = __alloy.HashMap.new()
```

A type import costs nothing at run time and follows the project's
existing `erase_type_imports` setting, as any other type import does.

The spec is `@alloy/std` rather than `@alloy` so the toolchain keeps
room for a second surface later, `@alloy/testing` for the assert
macros, for example, without giving one spec two meanings.

### The report

Under `globals = "none"`, a std name with no import reads as an unknown
name, with the import to write:

```
error(3.2): UnknownName: `HashMap` is a std type; write
`import { HashMap } from "@alloy/std"`
```

The message names the import rather than saying the name is unknown,
because the name is not a typo and the reader needs the line, not the
diagnosis. `alloy flux --fix` inserts it, and the editor offers it as a
quick fix beside the type auto-import quickfix that already exists.

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
- A per-file directive, `--@alloy-no-globals`, instead of a project
  setting. It makes the choice local, and it puts a compiler flag in
  every file of a project that wants it everywhere.
- Drop the ambient names outright, with no option. It is the cleaner
  language and it breaks every file written so far.
- Keep only the option to shadow, so a project's own `Signal` wins over
  the std one silently. That is the behavior this proposal exists to
  make visible.

## Prior Art

- Rust: a small prelude is ambient and everything else is a `use`. The
  list value here is that prelude, chosen per project rather than by
  the language.
- Python: no ambient library; every name is an import, and builtins are
  the fixed small set.
- Go: no ambient library, and an unused import is an error.
- Luau and Roblox: globals are ambient and a script cannot opt out,
  which is the state Alloy inherits and this proposal makes a choice.

## Unresolved questions

- Which names a project should keep ambient when it writes a list. The
  default stays `"all"`, so this is guidance for the docs rather than a
  change to the language.
- Whether an unused std import should draw the existing
  `unused_import` lint, which would make a copied file noisy while it
  is being adapted.
- Whether `@alloy/std` should also be importable under `"all"`, so a
  file may be explicit in a project that does not require it. Allowing
  it costs nothing and gives one name two spellings in one project.
- Whether the setting belongs in `[std]` or in the existing `[emit]`
  table, given that it changes name resolution and not the emit.
