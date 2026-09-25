# Luau parity and lighter headers

**Status**: Implemented

## Summary

A Luau file reads in Alloy the way Luau reads it, and five Alloy forms
now mean what they look like. The words `trait`, `impl`, `remote`,
`macro`, `attribute`, `namespace`, `private`, and `public` are names
away from their declarations. `[[ ... ]]` is a long string. A
declaration header needs no `as` when its body starts on the next line.
A match guard is written `where`. A catch-all arm counts toward an
exhaustive match, a value name in a pattern reports, and `in` on a
`{ }` literal with items reports. An arm can run statements before
its value, and `return` in it leaves the function. `@allow` also takes
the rustc and Clippy names of the lints Alloy has.

## Motivation

Alloy promises that a Luau file compiles. Two rules broke that promise
for ordinary Roblox code:

```luau
local remote = ReplicatedStorage.Remotes.Hit
local help = [[
Press E to open the shop.
]]
```

The first line reported "`remote` is a reserved word". The second did
not lex, because `[[` opened a nested array. A networking module names
a local `remote` in its first ten lines, and a UI module holds text in
long strings, so these two rules turned away the files a Luau user
tries first.

A header wrote a word that said nothing:

```alloy
struct Player as
    name: string
end
```

The line break already ends the header. Luau's own block headers take
no such word, and the Luau class proposal writes `class Point` with the
members below. Authors write the short form anyway: one game in the
test bed wrote all nine of its `impl` and `trait` headers with no `as`,
and each one was an error.

Four forms compiled to something the author did not mean, and three of
them reported nothing:

```alloy
const MAX = 10

match x with
    case MAX then "max"      -- binds a new MAX; the arm takes every value
    default "other"          -- never runs
end

print(2 in { 5, 6 })         -- true: 2 is a key of the table

match x with
    case 1 then "one"
    case _ then "other"      -- reported "not exhaustive"
end

match x with
    case n and n > 5 then    -- reads as one condition, `n and (n > 5)`
end
```

## Design

### The declaration words are names

Each word is a keyword where a declaration can start, and a name
everywhere else. The rule is the one `struct`, `enum`, and `interface`
already follow:

| word | a keyword before |
|---|---|
| `trait`, `impl`, `macro`, `attribute`, `namespace` | a name on the same line |
| `remote` | a name or `function` on the same line |
| `private`, `public` | a name or `function` on the same line |

```alloy
local remote = folder.Hit           -- a name
remote Hit(n: number) from client   -- a declaration
local private = {}                  -- a name
struct P
    private hp: number              -- a visibility word
    private: number                 -- a field named private
end
```

The formatter, the highlighter, and the language server read one
function, `keyword_at`, so the tools agree on each word.

### `[[` is a long string

`[[ ... ]]` and `[=[ ... ]=]` are Luau long strings. A nested array
puts a space between the brackets, and the formatter writes arrays
that way already:

```alloy
local grid = [ [1, 2], [3, 4] ]
```

After an intrinsic a string cannot follow, so `$map[["a", 1], ["b",
2]]` and `$set[[...]]` keep reading the bracket as the list of items.

### Headers

`as` is optional when the body starts on the next line or the header
closes with `end`:

```alloy
struct Player
    name: string
end

impl Named for Player
    function name(self): string return self.name end
end

enum Color as Red, Green, Blue end
```

A body on the header's own line keeps `as`, because the word is what
splits the two there. The rule holds for `struct`, `enum`, `trait`,
`interface`, `namespace`, and `impl`. An `attribute` body keeps `as`,
because an attribute without a body is a complete declaration.

`alloy fmt` removes an `as` that has the body below it, so a project
has one form. The editor indents after a header with no `as`. A body on
the header's line with no `as` still reports ``needs `as` before its
body``, and the quick fix writes it.

### Guards

A guard is written `where`, the word a `for` filter takes:

```alloy
match hit with
    case Hit(p, Circle(r)) where r > 10 then big(p)
    case Hit(p, _) then small(p)
end
```

`and` still parses. The style lint `match_guard_and` reports it, and
`alloy flux --fix` writes `where`. The spellings of other languages
report with the Alloy form: an `if` guard names `where`, `case A | B`
names `or`, and `Ok(v) => f(v)` names `case Ok(v) then`.

### Patterns

A binding or `_` after literal arms makes the match exhaustive. A
guarded arm still proves nothing.

A bare name in a pattern binds a new name. When the name is in
SCREAMING_CASE, the author meant a constant, so it is an error. A
lowercase local in scope is the lint `pattern_shadows_local`, because
binding over a local is sometimes meant:

```
error(4.2): ExhaustiveMatch: `MAX` names a value, and a bare name in a
pattern binds a new one, so this arm takes every value; compare in a
guard: `case n where n == MAX`
```

### Arms that run statements

An arm of a match that gives a value can run statements first. The
last line of the arm is its value, as in a Rust block:

```alloy
local label = match score with
    case 0 then "none"
    default
        print(score)
        tostring(score)
end
```

`return` leaves the nearest function body, `try do`, or `async do`. It
never leaves an arm. So in an arm, `return nil` leaves the function
around the match, and `break` and `continue` go to the loop around it.
One word keeps one meaning.

Luau has no expression that runs a statement. A match with such an arm
therefore stands in three places: after `local x =`, after `x =`, and
after `return`. The compiler moves each arm into that statement, and
every line stays where it is:

```luau
local label do local _m1 = score
    if _m1 == 0 then label = "none"
    else
        print(score)
        label = tostring(score)
end end
```

In any other place the match is an error that says to bind it to a
local first. An arm that ends in a statement other than `return`,
`break`, or `continue` reports `this arm gives no value`.

In a value block (an arm, `try do`, or `async do`), a line that starts
with `(`, `[`, `{`, `-`, or a string starts the value. It does not
continue the line above. Luau already refuses the `(` case as
ambiguous. The last line of a `try do` or an `async do` is its value
when it is an expression, and a call counts. Each of the two bodies runs
in a function of its own, so `...`, `break`, and `continue` inside it
are errors.

### `in` on a table literal

`x in t` searches a raw table by key. On a `{ }` literal with items the
keys are 1, 2, and so on, so the search is never what the author
meant. It is an error that names the array form:

```
`in` on a `{ }` literal searches its keys, 1, 2, and on, not its items;
write `[ 5, 6 ]` to search the items
```

A keyed literal, `kind in { rect = true, circle = true }`, is the Lua
set idiom and compiles.

### `@allow` names

`@allow` reads a rustc or Clippy lint name as the Alloy lint it means,
and `clippy.` is a tool prefix like `flux.`:

| written | quiets |
|---|---|
| `unused_variables` | `unused_variable` |
| `unused_imports` | `unused_import` |
| `dead_code` | `unused_function` |
| `needless_return` | `redundant_return` |
| `let_and_return` | `local_then_return` |
| `print_stdout`, `dbg_macro` | `print_debug` |
| `todo` | `todo_comment` |
| `clippy.too_many_arguments` | `too_many_arguments` |

## Drawbacks

- `[[1, 2], [3]]` was a nested array and is now a long string. No file
  in the examples, the ingots, or the two games in the test bed wrote
  it; the docs did, and they now write `[ [1, 2], [3] ]`. The lint
  `array_long_string` reports a long string whose text reads like
  arrays, and `alloy flux --fix` writes the array form.
- Two spellings of a header and of a guard. The formatter writes one
  header form, and the lint rewrites the old guard.
- In a value block a new line decides where a value starts: `-y`
  under a call is a value, not a subtraction. Luau reads a file the
  same with or without its line breaks, except for the `(` case it
  refuses.
- A match whose arms run statements stands in three places only.
  Inside a call it reports, and the fix is one local.
- A file that used one of the eight words as a name now compiles, so a
  typo such as `trait` alone on a line reads as a name and reports as
  an unknown global rather than a missing declaration.

## Alternatives

- Keep `as` required and let the quick fix write it. It keeps one form,
  and it keeps a word that tells the reader nothing.
- Remove `as` from one-line bodies too. `enum Color Red, Green end`
  reads as four names in a row.
- Keep `[[` as an array and ask for `[=[` in a long string. It breaks
  the promise that every Luau file parses.
- Compare when a pattern names a `const`, as Rust does. The reader of
  `case MAX` then has to know whether `MAX` is a const, a local, or a
  new name, and the case of one letter decides it. An error with the
  guard in it says what the line does.
- Keep one expression per arm. A value that needs a `print` first then
  needs a local function or an `if` chain, and the match stops being
  the one place the cases live.
- A word for the arm value, such as `yield v`. It adds a keyword for
  one place, and `yield` already names a coroutine yield in Luau.
- `return` as the arm value. `return` then means two things, and the
  reader of `return nil` has to find the nearest arm to know which.
- Wrap the match in a closure in every place. `return` in an arm would
  then leave the closure, and `break` and `continue` could not reach
  the loop.
- `if` as the guard word, as Rust writes it. `where` is already the
  filter word of `for`, `if local`, and `while local`, and a second word
  for one idea is a word to learn.

## Prior Art

- Swift writes a guard `case let x where x > 5`, the form this takes.
- Rust keeps `union` and `auto` as contextual keywords, and Luau does
  the same with `type`, `export`, and `continue`.
- A Rust block's value is its last expression, and `return` always
  leaves the function. Kotlin's `when` takes a block branch whose last
  expression is the value.
- Rust's `#[allow(clippy::lint)]` names a tool and its lint, the shape
  `@allow(clippy.lint)` follows.
- Rust warns when a pattern binds a SCREAMING_CASE name, through
  `non_snake_case`, and on the arms the binding makes unreachable.
  Alloy reports the case itself, as an error.
