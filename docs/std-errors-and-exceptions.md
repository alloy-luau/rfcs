# `Error`, `Exception`, `throw`, and `raise`

**Status**: Accepted

## Summary

The std gains two types and the language gains two words. `Error` is a
value a failure carries, built with `new Error("message")` and extended
by any struct that implements the `Error` trait. `throw e` raises it,
which `try do ... end` catches as an `Err`. `Exception` is the same
shape for a condition that must be seen and must not stop the caller,
and `raise e` reports one through `warn`.

## Motivation

Alloy already has one failure path and one hole in it.

The path is `Result`. A function that can fail returns `Result<T, E>`,
`try expr` returns the `Err` early, and `try do ... end` turns a block
into a `Result`. That works when the author owns the signature.

The hole is the value. `E` is any type, so today it is usually a
`string`:

```alloy
local function parse(text: string): Result<number, string>
    if #text == 0 then
        return Err("empty input")
    end
    ...
end
```

A string carries no kind, no fields, and no traceback. The caller who
wants to tell "empty input" from "not a number" compares text. A caller
who wants the line that failed has nothing. The `try do` doc already
promises "a throw becomes `Err` carrying the traceback from the throw
site", so the traceback exists inside the runtime and no type hands it
to the reader.

The second gap is the warning. Roblox code calls `warn` for a condition
the player survives: a missing asset, a deprecated call, a value that
fell back to a default. `warn` takes a string, so that report has no
kind either, and nothing can collect or count the conditions a session
raised.

## Design

### The `Error` trait and the `Error` struct

The std declares a trait, so any type can be an error:

```alloy
trait Error as
    function message(self): string
    function name(self): string
        return "Error"
    end
    function traceback(self): string?
        return nil
    end
end
```

and a struct that implements it, for the common case:

```alloy
export struct Error as
    message: string
    read name: string = "Error"
    read traceback: string? = nil
end
```

`new Error("empty input")` builds one. The constructor fills
`traceback` from the call site, so the value knows where it came from
before anyone throws it.

### Your own error types

Two ways, and the second is the one to teach.

A struct that implements the trait is an error:

```alloy
struct NotFound as
    key: string
end

impl Error for NotFound as
    function message(self): string
        return `no entry for {self.key}`
    end

    function name(self): string
        return "NotFound"
    end
end
```

A struct that holds an `Error` reuses its fields:

```alloy
struct HttpError as
    inner: Error
    status: number
end
```

Both answer `<T: Error>`, both match, and both cross `throw`. Neither
needs the class machinery, which does not compile yet.

### `throw`

```alloy
throw new Error("empty input")
throw new NotFound { key = id }
```

`throw` takes one operand that implements `Error`. It does not return,
so the compiler reads the code after it as unreachable, the same way it
reads the code after `return`.

Inside `try do ... end` a throw becomes the block's `Err`, which is
what the `try` doc already promises for a Luau `error`:

```alloy
local r = try do
    throw new NotFound { key = "abc" }
end
-- r is Err(NotFound)
```

Outside a `try`, a throw leaves the thread the way `error` does, and
the traceback the value carries names the throw site.

`throw` on a value that is not an `Error` is an error with a fix:

```
error(3.x): TypeError: `throw` takes an `Error`; write `throw new
Error("empty input")`
```

The fix wraps a string operand, since that is the mistake a reader of
other languages makes first.

### `raise`

```alloy
raise new Exception("asset 12345 is missing; using the default")
```

`raise` reports and continues. The value reaches `warn` with its name,
its message and its traceback, and the statement's own value is
nothing, so `raise` is a statement and not an expression.

`Exception` is the `Error` shape under another name, with its own trait
so the two never mix:

```alloy
export struct Exception as
    message: string
    read name: string = "Exception"
    read traceback: string? = nil
end
```

`throw` on an `Exception`, and `raise` on an `Error`, are each an error
that names the other word. The pair is the point: the type says which
one the author meant, and the compiler holds them apart.

### Emit

`throw`:

```luau
error(setmetatable({ ... }, Error), 0)
```

The level is `0`, because the value already carries the traceback from
its own construction, and Luau's own prefix would name the `throw` line
twice.

`raise`:

```luau
warn(__alloy.exception_text(value))
```

`exception_text` is one std function: it reads `name`, `message` and
`traceback` off the value and returns the line `warn` prints. The emit
stays on the source's own line, as every Alloy statement does.

`try do ... end` needs no change. Its `xpcall` already catches the
throw, and the std bridge already keeps `{ err = e, trace = ... }`. A
thrown `Error` arrives as the `err`, so the block's `Err` carries the
value and not a string.

### Catching by kind

A caught error is a value, so `match` reads it:

```alloy
match try do fetch(id) end with
    case Ok(row) then use(row)
    case Err(e) then
        match e with
            case NotFound { key } then warn(`missing {key}`)
            default warn(e:message())
        end
end
```

An `Error` in an `Err` keeps its own type, so a struct pattern narrows
it. That is the reason to have types rather than strings.

### The language server

- Completion after `throw ` offers the error types in scope, and
  `new Error(` first.
- Completion after `raise ` offers the exception types.
- A hover on `throw` and on `raise` says which type each takes and
  which one catches.
- The unreachable code after a `throw` greys the way it does after a
  `return`.

## Drawbacks

- Two words for one idea, and a reader of Python reads `raise` as the
  one that unwinds. It is the opposite here. This is the proposal's
  real cost, and the Alternatives name the other spellings.
- A second failure path beside `Result`. The rule that keeps them one
  path: a function that can fail in a way the caller should handle
  returns `Result`; a `throw` is for the failure no local caller can
  answer. `try do` is the bridge, so nothing is trapped on one side.
- `Error` is a name many codebases already use for a local or an
  import. It is a std export, so a file may shadow it, and a file that
  does gets its own meaning.
- `new Error(...)` builds a traceback on every construction, which
  costs a `debug.traceback` call on a path that may never throw.

## Alternatives

- `throw` alone, with no `raise`. `warn` already exists and takes a
  string. It loses the kind and the collectable report, which is the
  reason the second word is here.
- `panic` and `warn` as the two words. `panic` says "does not return"
  to a reader of Rust or Go, and `warn` is the Luau name already. This
  is the spelling to take if the Python reading of `raise` proves to be
  a real trip hazard in practice.
- `Error` as a `class`, per the classes RFC, with `extends` for the
  subtypes. It reads like TypeScript, which is what the request asked
  for. It waits on a feature that does not compile yet, and a trait
  already lets any struct be an error without a hierarchy.
- Nothing new: keep `Err("string")`. It works, and it is why no caller
  today can tell two failures apart without reading text.

## Prior Art

- JavaScript and TypeScript: `class Error`, `throw`, `try/catch`, and
  subclasses by `extends`. The request is shaped after this. Alloy's
  trait is the same idea without the hierarchy.
- Rust: `Result` plus the `Error` trait, and `panic!` for what no
  caller answers. The split proposed here is that split.
- Python: `raise` unwinds and `warnings.warn` reports. The word means
  the opposite of this proposal, which is the naming hazard above.
- Luau: `error(value, level)` takes any value, and `pcall` returns it.
  `throw` is that call with a type on the value.

## Unresolved questions

- Whether `throw` may take a bare string as sugar for
  `new Error(string)`. It reads well and it weakens the type rule that
  makes the feature worth having.
- Whether `raise` should also return the value, so a caller can count
  the conditions a session reported.
- Whether a `Result<T, E>` should bound `E` to `Error` by default. It
  would make the two paths one, and it would break every
  `Result<T, string>` written today.
- Whether an unhandled `throw` at the module top level should name the
  module in its report, the way the `try` doc says a top-level `try`
  yields the `Err` from the chunk.
