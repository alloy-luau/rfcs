# `|>`: pipe operator

**Status**: In development, behind the `AlloyExperimentalPipeOperator` flag

## Summary

`a |> f(_)` evaluates `a` once, binds it to `_`, and evaluates the right
side with it. A chain of pipes reads top to bottom in the order it runs,
and the value can go in any argument position or outside a call. Outside
the right side of a `|>`, `_` is an ordinary Luau name, as it is today.

## Motivation

A value passed through several free functions is written inside out:

```luau
local label = string.format("%d / %d", math.floor(math.clamp(humanoid.Health, 0, humanoid.MaxHealth)), humanoid.MaxHealth)
```

To read this, you start at `humanoid.Health` in the middle and work
outward through three calls. The format string ends up far from the
number it formats.

The usual fix is a local per step:

```luau
local health = math.clamp(humanoid.Health, 0, humanoid.MaxHealth)
local whole = math.floor(health)
local label = string.format("%d / %d", whole, humanoid.MaxHealth)
```

This reads in order, but it names intermediate values that are only
used once. Those names stay in scope for the rest of the function, so a
later line can use `health` when it meant the unclamped value.

Method chaining (`a:b():c()`) also reads in order, but it only works for
methods the value's type has. Most functions a Roblox project calls in
sequence are free functions in `math`, `string` and `table`, std
helpers, or UI libraries that take a composite as their first argument:

```luau
local c = ink.align(ink.tint(ink.text(scope, "HP"), palette.white, 0), Vector2.new(0, 1))
```

A wrapper object with those functions copied onto it as methods works
for one library at a time. It stops working when one step comes from a
different library.

## Design

### The form

```luau
expr |> body
```

`body` is any expression. Inside it, `_` is the value of `expr`:

```luau
local label = humanoid.Health
    |> math.clamp(_, 0, humanoid.MaxHealth)
    |> math.floor(_)
    |> string.format("%d / %d", _, humanoid.MaxHealth)
```

`_` can appear anywhere a name can be read, such as an argument, a
method receiver, an operand, a field access or a table field:

```luau
local alive = Players:GetPlayers()
    |> Array.filter(_, function(p) return p.Character ~= nil end)
    |> Array.map(_, function(p) return p.Character end)

local next_index = #items |> _ + 1
local name = model |> _:FindFirstChild("Humanoid") |> _?.DisplayName ?? "?"
local c = ink.text(scope, "HP")
    |> ink.tint(_, palette.white, 0)
    |> ink.align(_, Vector2.new(0, 1))
```

### The topic, `_`

`expr` is evaluated once, before `body`. If it returns several values,
`_` holds the first, as with a parenthesized call.

`_` is in scope for all of `body`, including function literals inside
it, which capture it as an upvalue. It cannot be assigned to inside
`body`. A `local _` or `for _, v in` inside a function literal in `body`
shadows it within that scope, like any other Luau local.

A pipe inside `body` binds its own `_` for its own body, and its left
side still sees the outer `_`. So `a |> f(_ |> g(_))` is `f(g(a))`.

`body` must read `_` at least once. `x |> f` is an error, not a call to
`f` (see Errors).

### Precedence

`|>` binds more loosely than every binary operator of Luau and of
Alloy, including `or` and `??`, and is left-associative:

```luau
a or b |> f(_)        -- (a or b) |> f(_)
x |> _ + 1 |> g(_)    -- (x |> _ + 1) |> g(_)
x |> _.name ?? "?"    -- x |> (_.name ?? "?")
```

The `else` branch of an `if ... then ... else` expression ends at the
next `|>`, so a pipe can follow the expression without parentheses, and
`if c then a else b |> f(_)` pipes the whole `if`. A position that has
its own end takes a whole pipe: an argument, a table field, an index,
the inside of parentheses, and the condition of an `if`.

### Multiple values

Each stage before the last passes one value on. The last stage's values
are the values of the whole pipe, so in `local a, b = x |> f(_)`, `a`
and `b` get both of `f`'s return values. A pipe in parentheses gives one
value, as a call in parentheses does.

### Emit

Alloy emits code on the same lines as the source. How a pipe is emitted
depends on where it appears.

A pipe is hoisted when it is the one expression on the right side of a
`local`, a `const`, or a `return`, or of an assignment to one target.
Each stage becomes a local on the line where that stage starts, and the
statement itself uses the last stage:

```luau
local label = humanoid.Health                            -- 1
    |> math.clamp(_, 0, humanoid.MaxHealth)              -- 2
    |> math.floor(_)                                     -- 3
    |> string.format("%d / %d", _, humanoid.MaxHealth)   -- 4
```

```luau
local __pipe1 = humanoid.Health                                  -- 1
local __pipe2 = math.clamp(__pipe1, 0, humanoid.MaxHealth)       -- 2
local __pipe3 = math.floor(__pipe2)                              -- 3
local label = string.format("%d / %d", __pipe3, humanoid.MaxHealth) -- 4
```

Each `_` is renamed to its stage's local. A stage that spans several
lines, such as one containing a function literal, keeps its lines. A
type annotation on the `local` (`local label: string = ...`) moves to the
last line along with the name. A `local` with several names keeps them
all on the last line, where they take the values of the last stage.

Hoisting runs the first stage before the statement it sits in. For a
`local`, a `const`, and a `return` nothing else runs first, so the order
does not change. An assignment reads its target first, so it hoists only
when that read has no effect: the target is a name, or one index of a
name by a field, a name, or a literal, such as `label`, `state.label`,
and `slots[i]`. Any other target, such as `find(id).label` or `a.b.c`,
takes the function form below, which keeps the order as written.

Anywhere else (an argument, an operand, a table field, or one of several
values), the stages are emitted the same way inside an immediately
invoked function, which is how the expression form of `match` is
emitted today:

```luau
print(humanoid.Health
    |> math.floor(_))
```

```luau
print((function() local __pipe1 = humanoid.Health
    return math.floor(__pipe1) end)())
```

If a body reads `...`, the function is declared as `function(...)` and
called with `(...)`.

The generated names use the compiler's reserved prefix, so they cannot
collide with names in user code.

### Errors

- `x |> f`: `the body of |> never reads _`, reported on `f`. When the
  body is a name or a field, the language server offers a fix that
  appends `(_)`.
- `x |>` with nothing after it: `expected an expression after |>`.
- `x |> function() _ = 1 end`: `_ is the piped value and cannot be
  assigned`, reported on `_`.
- A `|>` with the flag off: `` `|>` is not final; set
  `AlloyExperimentalPipeOperator = true` under `[fflags]` in alloy.toml
  to use it ``, reported on the `|>`.

Type errors inside a stage come from Luau and are reported on the
emitted local. Because that local is on the stage's line, the error maps
back to the stage, and `__pipeN` in the message is shown as `_`.

### The language server

- Hover on `_` shows the type of the value at that stage.
- Hover on `|>` shows the type of the value going into the stage.
- Completion after `_.` and `_:` is completion for that type.
- Rename and references treat `_` as a local of its stage. Renaming `_`
  itself is refused, since the author didn't choose that name.
- An inlay hint shows the type at the end of the line of each stage but
  the last, under the setting for the types of variables.

### The formatter

A pipe with more than one stage is formatted one stage per line, with
`|>` at the start of the line, indented one level past the start of the
statement. A single-stage pipe that fits within the line width stays on
one line.

## Drawbacks

`_` gets a second meaning. Inside a body it is the piped value, while
everywhere else Luau code uses it for values it ignores, so a reader
new to the feature may read `_` as "unused". Requiring the body to read
`_`, and making it read-only, keeps that case visible.

Code that reads a local named `_` from inside a body would read the
piped value instead. No existing code is affected, because `|>` does not
parse today, but code moved into a body can change meaning.

The expression-position form allocates a closure each time it runs, so
a pipe used as an argument in a hot loop costs the same as an
expression `match` does today. The statement form allocates nothing.

Each stage in statement position emits a local, and these count toward
Luau's limit of 200 locals per function. They also stay in scope until
the end of the block.

Stack traces and the debugger show `__pipeN` instead of `_`, though the
line number is still the stage's line.

People used to Elixir, F#, Gleam, R or PHP will expect `x |> f(a)` to
mean `f(x, a)` or `f(x)`. In Alloy that form fails to compile.

## Alternatives

First-argument insertion, as in Elixir and Gleam, makes `x |> f(a)` mean
`f(x, a)`. It is shorter for the common case and works well with
libraries designed for it. It is wrong for every function that takes
the value somewhere other than first, such as `string.format`,
`table.insert(t, _)` or `CFrame.new(_, target)`, and Roblox APIs are not
designed around putting the value first. The topic form handles both
cases at the cost of `_`.

Function application, as in F#, OCaml and PHP 8.5, makes `x |> f` mean
`f(x)`. It only fits one-argument functions. Luau functions are not
curried, so `x |> f(a)` would mean `f(a)(x)`, which Luau code almost
never wants.

An empty argument slot (`f(, a)`) was the first sketch of this proposal.
It only works for call arguments, cannot express `_ + 1` or `_.Name`,
and is easy to miss in review.

Other topic tokens each conflict with something. `%`, which the TC39
proposal uses, is already Luau's modulo operator, so `% % 2` would be
valid. `$` would be a new character for Luau, `@` is Alloy's attribute
sigil, and `#` is the length operator. `_` is already the conventional
Luau name for a placeholder value and needs no new token.

Wrapper objects (`Iter.from(t):filter(...):map(...)`) work when one library
provides every step. They don't help with `math`, `string`, or a UI
library's free functions.

Alloy macros are expression templates, so a macro cannot introduce a
binding that the caller's expression reads, which is what `_` needs.

A `do ... end` block around the stage locals of a hoisted pipe would end
their scope at the statement and give the 200 locals back. The declared
name would then be a `local` with no value and an assignment inside the
block. That form cannot write a `const`, it puts a `do` and an `end` on
lines the author did not write them on, and the checker may type a name
less exactly from a later assignment than from its declaration. The
stage locals stay in the block of the statement. A function that nears
the limit can put the pipe in a function of its own.

A bare name as a body, `x |> f` for `f(_)`, stays an error. It would be
the one body that does not read `_`, and `x |> f.g` and `x |> f()` would
then need a rule each. A later RFC can add it, since the form does not
compile today.

A call statement with a pipe as its one argument, `print(x |> f(_))`,
takes the function form and does not hoist. Hoisting there would run the
pipe before the callee, and `find(id):send(x |> f(_))` would change its
order. One rule covers every position: a pipe hoists only where nothing
of the statement runs before it.

If nothing is done, code stays nested inside out or names every
intermediate step.

## Prior Art

- Hack has `|>` with the topic `$$`, which the right side must use. This
  proposal follows Hack's rules.
- The TC39 proposal for JavaScript is at Stage 2. It is Hack-style with
  the topic `%`, and the token is still provisional. The topic must be
  used, is immutable, is lexically scoped so closures can capture it,
  and `|>` binds more loosely than nearly every operator. This proposal
  takes the same rules with `_` in place of `%`.
- Elixir inserts the value as the first argument.
- Gleam also inserts the value as the first argument. Separately, `_`
  in a call creates a function capture, so `f(a, _)` is a one-argument
  function and `x |> f(a, _)` puts `x` in the hole. That looks the same
  as this proposal for the common case, but in Gleam it builds a
  function, and the hole can only be a single argument of a single call.
- F#, OCaml and Elm use `x |> f` for `f(x)`, relying on curried
  functions. Julia's `|>` also applies a one-argument function.
- R added `|>` in 4.1, inserting the value as the first argument, and
  added a `_` placeholder in 4.2 that can only be used as a named
  argument.
- PHP added `|>` in 8.5. The right side must be a one-parameter
  callable.
- Luau has no pipe operator, and no proposal for one is on record.
  On 2026-11-16 no pull request of luau-lang/rfcs, open or closed,
  named a pipe in its title, and a search of that repository and of the
  issues of luau-lang/luau for `pipe operator`, `pipeline operator`, and
  `|>` found nothing. So Luau has not turned the operator down, and it
  has not taken the `|>` token.

- In Alloy, `?.` chains through a value's own fields, and the expression
  form of `match` already emits an immediately invoked function with a
  hoisted local. This proposal uses the same emit and the same reserved
  prefix.
