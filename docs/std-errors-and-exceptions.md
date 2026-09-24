# `Error`, `Exception`, and `throw`

**Status**: Accepted

## Summary

The std gains two types and the language gains one word. `Error` is a
value a failure carries, built with `new Error("message")` and extended
by any struct that implements the `Error` trait. `throw e` raises it,
which `try do ... end` catches as an `Err`. `Exception` is the same
shape for a condition that must be seen and must not stop the caller.
`warn(e)` reports one: its `__tostring` writes the name, the message,
and the traceback, so Luau's own `warn` needs no new word beside it.

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
trait Error
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
export struct Error
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
struct NotFound
    key: string
end

impl Error for NotFound
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
struct HttpError
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

### Reporting an `Exception`

```alloy
warn(new Exception("asset 12345 is missing; using the default"))
```

`warn` reports and continues, as it does today. `Exception` carries a
`__tostring`, so the line `warn` prints holds the name, the message and
the traceback. No keyword is needed: the call is the one Roblox code
already writes, and a value that is not an `Exception` prints the way
it always did.

`Exception` is the `Error` shape under another name, with its own trait
so the two never mix:

```alloy
export struct Exception
    message: string
    read name: string = "Exception"
    read traceback: string? = nil
end
```

`throw` on an `Exception` is an error that names `warn`. The type says
which one the author meant, and the compiler holds them apart.

### Emit

`throw`:

```luau
error(setmetatable({ ... }, Error), 0)
```

The level is `0`, because the value already carries the traceback from
its own construction, and Luau's own prefix would name the `throw` line
twice.

`warn(e)` emits as written. `Exception.__tostring` is one std
function: it reads `name`, `message` and `traceback` off the value and
returns the line `warn` prints.

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
- A hover on `throw` says which type it takes and which one catches.
- The unreachable code after a `throw` greys the way it does after a
  `return`.

## Drawbacks

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

- A `raise` statement for the `Exception`, beside `throw`. It was the
  first draft. A reader of Python reads `raise` as the word that
  unwinds, the opposite of what it did here, and `warn` already reports
  a value whose `__tostring` writes the report.
- `panic` for the word that unwinds. It says "does not return" to a
  reader of Rust or Go. `throw` pairs with `try` and `Error` the way a
  TypeScript reader knows them, and the request came from that reader.
- `throw "text"` as sugar for `throw new Error("text")`. It reads well,
  and it weakens the type rule that makes the feature worth having. The
  report's fix writes the wrapper instead.
- `Result<T, E>` bounding `E` to `Error` by default. It would make the
  two paths one, and it would break every `Result<T, string>` written
  today.
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
- Python: `raise` unwinds and `warnings.warn` reports. The first draft
  of this proposal used `raise` to report, the opposite reading, which
  is why the word is gone.
- Luau: `error(value, level)` takes any value, and `pcall` returns it.
  `throw` is that call with a type on the value.
