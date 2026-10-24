# Parallel Luau: actor scripts, `parallel` blocks, and typed messages

**Status**: Implemented

Implemented in 0.1.0-rc.10. This text describes what the compiler
builds. A "Changed in the build" note marks each place where the build
differs from the first draft.

## Summary

Four pieces, each small on its own. A file named `x.server.actor.aly`
or `x.client.actor.aly` builds into an `Actor` that holds the script,
the way `.server.` and `.client.` already pick a script class. A
`parallel do ... end` block runs desynchronized, and the compiler
refuses the writes the engine refuses. A `message` declaration is a
typed channel between actors, the way `remote` is a typed channel
across the network. A `reply(...)` clause gives a message a typed
answer that any script can take.

## Motivation

Roblox runs a script in parallel when the script sits under an `Actor`,
and the code asks for the parallel phase with `task.desynchronize()`
and returns with `task.synchronize()`. The engine is strict about what
a parallel phase may touch, and the failure mode is a run-time error in
a frame that is hard to reproduce.

Writing it by hand today costs three things.

The placement is not in the source. An `Actor` is an instance, so it
lives in the Rojo project file or in a `.model.json`, far from the code
that needs it. A reader of the file cannot tell whether it runs in
parallel.

The phases are unchecked. `task.desynchronize()` and
`task.synchronize()` are plain calls, so nothing stops a write to a
part's `Position` between them, and nothing says which functions are
safe to call from the parallel phase.

The messages are strings. The engine's API is
`SendMessage(topic: string, ...: any)` and
`BindToMessage(topic: string, func: (...any) -> ...any)`, both
confirmed in the definitions Alloy already ships. A typo in a topic is
silent, and every argument is `any` on both sides. Alloy already solved
that exact shape for the network with `remote`, which generates the
serializer and types both ends from one declaration.

## Design

### The script kind

`.actor.` after the side puts a script in an `Actor`. The file
`src/server/physics.server.actor.aly` builds as the Actor `physics`
with the Script `physics` in it, and `.client.actor.aly` makes one with
a LocalScript. The script keeps the class that its side gives it. The
infix follows the convention Alloy already reads, so a reader who knows
`.server.` needs no new rule and no project file edit.

The build writes the script as `physics/physics.server.luau`, beside an
`init.meta.json` that gives the folder the class `Actor`:

```json
{
  "className": "Actor"
}
```

Rojo reads that file, so the project file and the source cannot
disagree. The sourcemap holds the Actor too, so `script.Parent` is the
Actor in the editor. A rename back to `physics.server.aly` takes the
Actor folder away on the next build.

The script sits one folder deeper in the game than its file sits in
`src`, so a relative import climbs one more folder in the emit:
`import { f } from "./util"` emits `require("../util")`. `alloy flux`
and the editor place the script in its Actor too, in a project with a
Rojo file and in a project without one.

Only a script takes the infix. A `ModuleScript` under an `Actor` runs
nothing on its own, so `.actor.` on a module is an `ActorError` that
names the script to put it on. An `init` script takes the name of its
folder, so it takes no infix either. `x.actor.server.aly` puts the
infix before the side, and the report gives the order.

**Changed in the build**: the first draft named the file `x.actor.aly` and
said the build adds the `Actor` at the leaf of the tree. The build
needs the side before `.actor.`, and it writes the Actor as a folder
with a meta file, which Rojo reads with no project file edit.

One file gives one `Actor`. The infix takes no count. A pool is a
tuning number that changes with the device and the load, and a number
in a file name can only be changed by renaming the file. A pool is
built at run time from the one the build writes, which is what the
engine's own pattern does:

```alloy
const template = script.Parent->physics :: Actor

for i = 2, 4 do
    local worker = template:Clone()
    worker.Name = `physics{i}`
    worker.Parent = template.Parent
end
```

### `parallel do ... end`

```alloy
const parts: { BasePart } = {}

local function settle()
    const hits: { BasePart } = {}

    parallel do
        -- Reads are allowed here. Instance writes are not.
        for _, part in parts do
            if part.Position.Y < 0 then
                table.insert(hits, part)
            end
        end
    end

    for _, part in hits do
        part.Position = new Vector3(0, 10, 0)
    end
end
```

The block emits the two calls on the lines of its header and its
`end`, so the line count holds, and the block keeps its own scope:

```luau
    task.desynchronize() do
        -- Reads are allowed here. Instance writes are not.
        for _, part in parts do
            if part.Position.Y < 0 then
                table.insert(hits, part)
            end
        end
    task.synchronize() end
```

**Changed in the build**: the first draft put each call on a line of its
own. The build adds `do` and `end` around the body, so a local that
the block declares ends with the block, as the source reads.

A `parallel` block is a statement, not an expression: the phase is a
property of the thread, not a value.

What the compiler refuses inside the block:

- A write to `Parent`, and a write to any other property of an
  `Instance` when the file knows the receiver's type. The file knows it
  for:
  - a name annotated with an Instance class, and a Roblox global such as
    `workspace` or `game`, with any chain of fields under it;
  - a local that `Instance.new` built, a value cast to an Instance class
    with `::`, and the result of `GetService`, `FindFirstChild`, `->`, or
    `=>`;
  - an element of a table typed with an Instance class, by index or in a
    `for` loop, `GetChildren` included;
  - a struct field typed with an Instance class;
  - a parameter of an `as parallel` message handler, with the type its
    message gives it.
- A call to `Instance.new`, `new Instance`, `Destroy`, or `Clone`, with
  `:` or with `.`, and the `destroy` and `delete` statements.
- A call to a function or a struct method the file declares that does
  one of those, at any depth. The report names the chain of calls. The
  check stays inside the file, because a whole program analysis is a
  different proposal.
- A nested `parallel` block, which the engine reads as already
  desynchronized.
- `await`, because a Future resumes in the serial phase, and the block
  would end somewhere the author did not write.
- A `return`, and a `break` or `continue` that leaves the block,
  because the thread would skip `task.synchronize()`. A `break` of a
  loop inside the block stays inside it, and the block allows it.

**Changed in the build**: the last item is new. The build also refuses
`new Instance` and the `destroy` and `delete` statements, which are
Alloy's own spellings of the calls the draft named.

**Changed in 0.1.2**: the check follows helpers at any depth, and it
knows the receivers in the first item that are not a plain annotated
name. A local or a loop variable that shadows a typed name is a new
value, and it does not take the outer name's type.

Each report names the engine's rule and the write it refuses. The
kind is `ParallelError`, section 3.15 of the book. A write of
`part.Position` inside the block, where `part` is a parameter typed
`BasePart`, reports:

```
worker.server.actor.aly:3:9 error(3.15): ParallelError: a `parallel` block cannot write `part.Position`; move the write after the block
```

The check reads this file and no further. A call into another module
is not followed, even one the project owns. A partial cross-module
check would be less useful than a line a reader can hold: the compiler
sees what the file says, and the engine has the last word. Whole
program analysis is a separate proposal.

So this is a guard rail and not a proof. A write through a value typed
`any`, a table with no type, or a value from another module cannot be
seen, and does not report. The `parallel` entry of the docs says so in
its text, not in a footnote.

`parallel` composes with `@cfg` and needs no rule of its own. A
`@cfg(server)` on a statement guards whatever the statement is, a
`parallel` block included, and the engine's phase rules are the same on
both sides. The side a script runs on is already decided by the
`.server.` or `.client.` infix beside `.actor.`, so there is nothing
for a parallel block to decide again.

A `parallel` block runs in any script under an Actor. The compiler
does not ask for the infix on the file that holds the block, because a
module that an actor script requires runs under that Actor too.

### `message`, a typed channel between actors

```alloy
--- One tick of the simulation.
export message Step(dt: number, gravity: number)
```

One declaration gives both ends, the way `remote` does. The examples
here use three files: the messages in `src/shared/physics.aly`, the
worker in `src/server/physics.server.actor.aly`, and the script that
owns the worker in `src/server/main.server.aly`. Each script imports
the messages it uses, as in
`import { Scan, Step } from "@shared/physics"`.

`on` and `once` bind on the Actor of the script that calls them, so
they stand in the worker:

```alloy
Step.on(function(dt, gravity)
    print(dt * gravity)
end)
```

`fire` sends to the Actor it takes first, from any script. The owner
finds the Actor that the build wrote beside it:

```alloy
const worker = script.Parent->physics :: Actor

Step.fire(worker, 0.016, 196.2)
```

The emit is the engine's own API with the topic and the arguments
filled in. The declaration emits a table with `topic` and `fire`, and
each call writes the engine call in place:

```luau
local Step = { topic = "Step", fire = function(actor: Actor, ...) actor:SendMessage("Step", ...) end }
worker:SendMessage("Step", 0.016, 196.2)
script:GetActor():BindToMessage("Step", function(dt, gravity)
```

`once` binds for the first message only. The engine has no one-shot
bind, so the emit calls a std helper that disconnects the binding and
then runs the handler.

`as parallel` binds with `BindToMessageParallel` instead, so the
handler runs in the parallel phase and takes the same rules a
`parallel` block takes:

```alloy
--- A part to check. The handler runs in the parallel phase.
export message Check(part: BasePart) as parallel
```

```alloy
Check.once(function(part)
    print(part.Position)
end)
```

A write that the handler cannot make reports with the advice to call
`task.synchronize()` before it. A `task.synchronize()` call that the
handler body holds directly ends the parallel phase for the check, and
the statements after it run serial. A call inside a branch or a loop
may not run, so the check goes on past it.

The parameters type both ends, so a wrong argument is a build error
rather than a nil at run time. The topic is the declaration's name, so
a typo cannot compile. `Step.on` in a file with no `.actor.` infix is
an `ActorError` that gives the name to rename the file to. In such a
file, completion after `Step.` does not offer `on` and `once`.

**Changed in the build**: in the first draft, a handler sent a message
back with no Actor, `Result.fire(hits)`, and the script that owns the
actor bound it with `on`. In the build, `fire` always takes the target
Actor first, and `on` and `once` stand only in an actor script. A
script under no Actor takes an answer through `reply`, below.

A message carries what the engine carries: the arguments cross as they
are, with no serializer, since an actor boundary is not a network
boundary. A function, a coroutine, and a value that keeps its methods
on a metatable, such as a struct, a payload enum, or a `HashMap`, do
not arrive whole. Each one is a `WireType` report with the rule named
for actors. A parameter takes no default and no pattern, because the
engine passes what `fire` sends.

An attribute goes on a message when its declaration names `message` in
its targets. A contract reads the parameters as fields:

```alloy
--- A message that names its job.
attribute numbered on message as
    requires field job: number
end

--- A job, and the levels a worker answers with.
@numbered
export message Light(job: number, input: buffer) reply(job: number, levels: buffer)
```

The name holds the table of the declaration, so
`Attributes.get(Light, numbered)` reads the attribute at run time.
`on` and `once` bind only through the name, because the compiler
writes them on the Actor of the script where they stand.

**Changed in the build**: the first draft had no attributes on a message.

### `reply`, an answer to any script

```alloy
--- A scan of the parts. The worker answers with the parts that fell.
export message Scan(parts: { BasePart }) reply(fallen: { BasePart })
```

`reply(params)` gives the message an answer. The handler of `on` and
`once` takes `respond` after the parameters, typed from the reply:

```alloy
Scan.on(function(parts, respond)
    const fallen: { BasePart } = {}

    parallel do
        for _, part in parts do
            if part.Position.Y < 0 then
                table.insert(fallen, part)
            end
        end
    end

    respond(fallen)
end)
```

`replied` binds a handler of the answer. It stands in any script,
under an Actor or not, so the script that owns the worker takes the
answer there:

```alloy
Scan.replied(function(fallen)
    print(#fallen)
end)
```

```alloy
Scan.fire(worker, workspace:GetChildren() :: { BasePart })
```

The emit gives the message a table that the std builds, and wraps the
handler so that it takes `respond`:

```luau
local Scan = __alloy.message("Scan", 1)
script:GetActor():BindToMessage("Scan", __alloy.message_respond(function(parts, respond)
```

The header reads the parameters, then `reply(...)`, then
`as parallel`.

The answer rides a `BindableEvent`. Each Luau VM makes its own on its
first `replied`, and `fire` sends it after the arguments, where the
source does not show it. A `fire` before any `replied` of its VM sends
no event, and `respond` then does nothing, so bind `replied` before the
first `fire`.

A `BindableEvent` fires only in the serial phase, so `respond` in an
`as parallel` handler calls `task.synchronize()` first, and the
statements after it run serial. A `respond` that the handler body
holds directly ends the parallel phase for the check, as a
`task.synchronize()` call does.

Nothing matches an answer to its request. Every `replied` handler of
the message in the VM that fired takes every answer, so pass a job
number both ways, as `job` does in `Light` above.

A reply parameter takes the rules of a message parameter. `respond`
may leave out the optional values at the end of the reply, and passes
no more than the reply holds. A message whose parameters end in `...`
takes no reply, because the handler takes `respond` after the
parameters and `...` has no end.

**Changed in the build**: the first draft had no answer. A script under no
Actor could not take a message back, because `on` binds on the Actor
of its own script. `reply` closes that gap.

### Shared state

A message is the channel this proposal gives. `SharedTable` stays what
it is today, a std type the author reaches for directly, because its
semantics are the engine's and wrapping them would hide the cost. The
docs point at it from the `parallel` entry.

A `SharedTable` crosses a message like any other value, with no special
handling. It is already shared by reference, so there is nothing to
copy and nothing to serialize, and giving it a rule of its own would
suggest Alloy manages its lifetime. It does not.

### The language server

- Completion after `parallel ` offers `do`. After the parameters of a
  message it offers `reply` and `as parallel`, and after `as` it
  offers `parallel`.
- A hover on `parallel` says which phase the code runs in, for a block
  and for `as parallel`.
- A write the block refuses draws its error on the write. The quick
  fix "Move the write after the block" moves the statement after the
  block when nothing later in the block reads what it writes, the
  statement is not in a loop, and it uses no local of the block.
- A message name completes `topic` and `fire`, `replied` when the
  message has a reply, and `on` and `once` in an actor script, typed
  from the declaration, the way a `remote` does. A message hovers as
  its declaration with the comment above it.
- Signature help lists the parameters for `fire`, for the handler of
  `on` and `once` with `respond` last, for `replied`, and for
  `respond`.
- `@` above a message offers the attributes that take `message` as a
  target.

## Drawbacks

- One more file-name infix. `physics.server.actor.aly` is long, and a
  reader has to know two conventions rather than one.
- Three new words: `parallel`, `message`, and `reply`. Each one stays
  free as a name. `parallel` opens a block only before `do` on the
  same line, `message` declares only before a name and `(`, and
  `reply` opens its clause only after the parameters of a message.
  **Changed in the build**: the first draft gave up `parallel` as a name
  that starts a statement.
- The write check is partial. An author may read it as a guarantee and
  meet the engine's error anyway. The wording of the docs carries that
  weight, which is a weak place to carry it.
- A message adds a declaration kind that looks like `remote` and is not
  one: no serializer, no side, a different failure mode. Two similar
  shapes with different rules is a real cost.
- A reply goes to every `replied` handler of the message in the VM
  that fired, and nothing pairs an answer with its request. The author carries a job
  number to do that.

## Alternatives

- An attribute, `@actor` on the file's top-level, instead of the file
  name. It puts the fact in the source where a reader sees it, and it
  splits placement between the name (`.server.`) and an attribute,
  which is worse than one convention.
- A `[actors]` table in `alloy.toml` naming the files. Configuration
  away from the code, which is the state this proposal improves.
- `parallel` as a function modifier, `parallel function step()`, rather
  than a block. It reads well and it makes the whole body parallel,
  which is rarely what the author wants; the useful shape is a hot loop
  inside a serial function.
- Nothing for messages: let the author call `SendMessage` directly.
  That works, and it leaves the topic a string and every argument
  `any`, which is the gap this closes.

## Prior Art

- Roblox parallel Luau: `Actor`, `task.desynchronize`,
  `task.synchronize`, `SendMessage`, `BindToMessage`,
  `BindToMessageParallel`, `SharedTable`. This proposal is a spelling
  for those, not a new runtime.
- Rust: `Send` and `Sync` decide what crosses a thread, checked by the
  compiler. The write check here is the same intent with far less
  reach, and the docs should not claim otherwise.
- Erlang and Go: typed messages between processes, which is the
  `message` declaration's shape.
- Alloy: `remote`, which already turns one declaration into a typed
  channel with both ends generated.
