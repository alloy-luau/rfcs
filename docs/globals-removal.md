# Remove `global`

**Status**: Accepted

## Summary

`global` leaves the language. A `global local`, `global const`, `global
function`, `global struct`, `global type`, `global namespace`, and
`global attribute` each become an error that names the `import` to
write. The side rules that exist only for globals leave with it:
`--@alloy-side`, `--@alloy-file-side`, the `[contexts]` table, the
shared-context detection, and the `mutable_global` lint.

## Motivation

A global is a name every file sees with no `import`. Luau has modules
for that, and Roblox warns against `_G` for the same reasons this
proposal gives.

The cost showed up as bugs. In one week the feature drew seven: a
`global local` that each file copied at `require` time, so a write in
one file reached no other; an inferred type that did not cross files; a
`const` member of a namespace that any file could assign; a global in a
`typeof` slot that the rewrite missed; emit temps `_g1` and `_gs` in the
completion list; a path comparison that reported a global as its own
duplicate; and the side rules interacting with each of these. Every
feature that reads names across files paid a tax for globals, and
every future one would too.

What a global buys is one line. The emit already lowers a global to a
module every file requires, so `global local counter = 0` in `a.aly` and
`counter` in `b.aly` is `import { counter } from "./a"` with the line
left out. Auto-import writes that line.

The user's own read, on the day of the decision: modules exist for this;
`_G` is warned against; a global is to a module what `goto` is to a
loop.

## Design

### What a user writes

Nothing. The forms that remain cover every use:

```alloy
-- a.aly
export local counter = 0
export const LIMIT = 5
export function bump() counter += 1 end

-- b.aly
import { counter, LIMIT, bump } from "./a"
```

A value that several files share is an `export local` in one module,
read and written through the functions that module exports. A bare
imported name is a copy taken at `require` time, as in Luau, so a
write to it in another file reaches no one; `bump()` and `read()`
above are the shape that shares. The `global local` emit rewrote every
use to the module's slot; nothing replaces that, and the doc for
`global` says so.

### The error

A `global` declaration reports at the keyword:

```
error(3.x): ImportError: `global` is removed; declare `counter` with
`export local` and write `import { counter } from "./a"` where it is
read
```

A use of a name that was global in another file reports as an unknown
name, which it is, with the auto-import action that already exists for
any unresolved name.

### The code action

On the declaration: "replace `global` with `export`" rewrites the one
word. On a use in another file: the existing auto-import action writes
the `import` line. Together they migrate a project in two clicks per
name.

### What leaves

- `alloy/src/globals.rs`, the index of every global in a project.
- The `global_prologue` in the desugar, the `_g1`/`_gs` slot emit, and
  the `typeof` rewrite for a global.
- The side rules: `--@alloy-side`, `--@alloy-file-side`, `[contexts]`,
  the ReplicatedFirst and ReplicatedStorage detection, and the
  client/server/shared placement check.
- The `mutable_global`, `shadowed_global`, and duplicate-global reports.
- The proxy's global hover, completion, and definition paths.
- `alloy doc global` and every doc line that names the form.

### What the emit does

Nothing new. A file with no `global` compiles as it does today. The
line count holds because no line changes.

### The language server

An `import` completes, hovers, and navigates as it does today. A
`global` keyword completes to nothing and hovers as the removal notice
with the two forms that replace it.

## Drawbacks

A project that uses `global` stops compiling until it is migrated. The
error names the fix and the code action applies it, and the feature is
in a release candidate, so the cost lands on few files.

A cross-cutting constant now needs one `import` per file that reads
it. Auto-import writes it, and the line says where the value lives,
which the global form hid.

## Alternatives

**Keep `global` as sugar for an implicit import.** That is what it is
now, and it still needs its own resolution path in the compiler and
the server, which is where the bugs were.

**Keep `global const` and `global type`, drop `global local`.** The
mutable form drew the worst bugs, but the resolution tax is the same
for every kind, and a rule with an exception is harder to teach than
none.

**Keep the side rules for imports.** `--@alloy-side` and `[contexts]`
were written for globals. An import already says which module it
reads, so a side rule on top says nothing the path does not.

## Unresolved questions

- Whether `export local` should carry a lint of its own for mutable
  shared state, as `mutable_global` did. The default is no lint; a
  shared slot behind an explicit `import` is what a module is for.
