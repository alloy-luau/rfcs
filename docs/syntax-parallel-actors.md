# Parallel Luau: actor scripts, `parallel` blocks, and typed messages

**Status**: Accepted

## Summary

Three pieces, each small on its own. A file named `x.actor.aly` builds
into an `Actor` that holds the script, the way `.server.` and
`.client.` already pick a script class. A `parallel do ... end` block
runs desynchronized and the compiler refuses the writes the engine
refuses. A `message` declaration is a typed channel between actors, the
way `remote` is a typed channel across the network.

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

```
src/systems/physics.server.actor.aly
```

`.actor.` in a file name puts the script inside an `Actor` named for
the file, and the script keeps the class the other infix picks, so
`.server.actor.` is a `Script` under an `Actor` and `.client.actor.` is
a `LocalScript` under one. The infix follows the convention Alloy
already reads, so a reader who knows `.server.` needs no new rule and
no project file edit.

The build writes the `Actor` into the tree it already writes, at the
mount the folder names. A project with a Rojo file still works: Alloy
reads the tree as written and adds the `Actor` at the leaf, so the
project file and the source cannot disagree.

Only a script kind takes the infix. A `ModuleScript` under an `Actor`
is meaningless on its own, so `.actor.` on a module is an error that
says to put it on the script that requires the module.

One file gives one `Actor`. The infix takes no count. A pool is a
tuning number that changes with the device and the load, and a number
in a file name can only be changed by renaming the file. A pool is
built at run time from the one the build writes, which is what the
engine's own pattern does:

```alloy
for i = 1, workers do
    local worker = template:Clone()
    worker.Parent = folder
end
```

### `parallel do ... end`

```alloy
local function step(dt: number)
    local hits = {}

    parallel do
        -- Reads are allowed here. Writes are not.
        for _, part in parts do
            if part.Position.Y < 0 then
                table.insert(hits, part)
            end
        end
    end

    for _, part in hits do
        part.Position = Vector3.new(0, 10, 0)
    end
end
```

The block emits the two calls on its own lines, so the line count
holds:

```luau
task.desynchronize()
for _, part in parts do ... end
task.synchronize()
```

A `parallel` block is a statement, not an expression: the phase is a
property of the thread, not a value.

What the compiler refuses inside the block:

- A write to a property of an `Instance`, when the receiver's type is
  known. `part.Position = v` reports; a write through a value typed
  `any` cannot be seen and does not.
- A call to `Instance.new`, `Destroy`, `Clone`, or a parent write.
- A call to a function the file declares that itself writes an
  instance. One level deep, from the same file, because a whole
  program analysis is a different proposal.
- A nested `parallel` block, which the engine reads as already
  desynchronized.

Each report names the engine's rule rather than the emit:

```
error(3.x): ParallelError: a `parallel` block cannot write
`part.Position`; move the write after the block
```

The check reads this file and no further. A call into another module
is not followed, even one the project owns. A partial cross-module
check would be less useful than a line a reader can hold: the compiler
sees what the file says, and the engine has the last word. Whole
program analysis is a separate proposal.

So this is a guard rail and not a proof, and the docs must say that
plainly rather than in a footnote.

`await` inside a `parallel` block is an error: a Future resumes on the
serial phase and the block would end somewhere the author did not
write.

`parallel` composes with `@cfg` and needs no rule of its own. A
`@cfg(server)` on a statement guards whatever the statement is, a
`parallel` block included, and the engine's phase rules are the same on
both sides. The side a script runs on is already decided by the
`.server.` or `.client.` infix beside `.actor.`, so there is nothing
for a parallel block to decide again.

### `message`, a typed channel between actors

```alloy
message Step(dt: number, gravity: number)

message Result(hits: Part[]) as parallel
```

One declaration gives both ends, the way `remote` does:

```alloy
-- In the actor's script.
Step.on(function(dt, gravity)
    ...
    Result.fire(hits)
end)

-- In the script that owns the actor.
Step.fire(actor, 0.016, 196.2)
Result.on(function(hits) ... end)
```

The emit is the engine's own API with the topic and the arguments
filled in:

```luau
actor:SendMessage("Step", 0.016, 196.2)
script:GetActor():BindToMessage("Step", function(dt, gravity) ... end)
```

`as parallel` binds with `BindToMessageParallel` instead, so the
handler runs in the parallel phase and takes the same rules a
`parallel` block takes.

The parameters type both ends, so a wrong argument is a build error
rather than a nil at run time. The topic is the declaration's name, so
a typo cannot compile. `Step.on` in a script with no `Actor` above it
is an error naming the `.actor.` infix.

A message carries what the engine carries: the arguments cross as they
are, with no serializer, since an actor boundary is not a network
boundary. A value the engine refuses to pass gets the report the wire
types already give, with the rule named for actors.

### Shared state

A message is the channel this proposal gives. `SharedTable` stays what
it is today, a std type the author reaches for directly, because its
semantics are the engine's and wrapping them would hide the cost. The
docs point at it from the `parallel` topic.

A `SharedTable` crosses a message like any other value, with no special
handling. It is already shared by reference, so there is nothing to
copy and nothing to serialize, and giving it a rule of its own would
suggest Alloy manages its lifetime. It does not.

### The language server

- Completion after `parallel ` offers `do`.
- A hover on a `parallel` block says which phase the code runs in and
  links the rule.
- A write the block refuses draws its error on the write, with a quick
  fix that moves the statement after the block when the value it
  computes is not read inside it.
- A `message` name completes its `fire`, `on` and `once`, typed from
  the declaration, the way a `remote` does.

## Drawbacks

- One more file-name infix. `physics.server.actor.aly` is long, and a
  reader has to know two conventions rather than one.
- `parallel` becomes a word that cannot be a statement-leading name.
  It stays free elsewhere, as `match` and `try` are.
- The write check is partial. An author may read it as a guarantee and
  meet the engine's error anyway. The wording of the docs carries that
  weight, which is a weak place to carry it.
- A message adds a declaration kind that looks like `remote` and is not
  one: no serializer, no side, a different failure mode. Two similar
  shapes with different rules is a real cost.

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
