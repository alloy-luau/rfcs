Background
===

Alloy is a strict superset of Luau that compiles to Luau on the same
lines. A change to the language therefore lands in three places at once:
the syntax a reader writes, the Luau the compiler emits, and the answers
the language server gives about both. Every one of them is a promise to
somebody, so a user facing change goes through a written proposal first.

When new syntax is introduced, we ask:

- Does every Luau file still parse and mean what it meant?
- Does the emit stay on the source's own lines?
- Does the shape read to a person who knows Luau but not Alloy?
- Does it create an ambiguity for the syntax we have or the syntax we
  may want?
- Can the language server complete it, hover it, and place an error on
  the token that caused it?
- Does the formatter have one form for it, and does that form hold when
  it runs twice?

For a change in meaning, we ask:

- Is the behaviour easy to hold in the head, and free of surprise?
- Does the emit stay something a reader of the output recognizes?
- Does Luau's own checker type it, and does a wrong use read as a
  message about what the author wrote?
- Does it hold on the client and on the server, and inside markup?

For an addition to the std, we ask:

- Does the code that needs it exist today, often enough to carry a name?
- Does the runtime earn its place, or does user code do it as well?
- Is the behaviour general, rather than one project's shape?
- Does the type read in a hover, and does the wrong argument report
  plainly?

Every addition carries a cost. A language is harder to learn for each
word it holds, slower to implement well, and richer in the ways its
features surprise each other. A decision here is expensive to reverse,
since code in the wild depends on it, so the discussion happens before
the work does.

Process
===

To open an RFC, open a pull request that adds one Markdown file to
`docs/`. The file follows `TEMPLATE.md`. Name it for the change, in
lowercase letters, digits, and dashes, with the area first:
`syntax-pipe-operator.md`, `std-string-split.md`,
`config-lint-groups.md`, `emit-remote-batching.md`.

Every open RFC stays open for at least two weeks, so there is time to
read it and to raise a concern. The discussion belongs in the pull
request. A point made elsewhere is summarized there, or it did not
happen.

When the two weeks are up, the RFC merges if the change is worth making
and the design as written is workable. A revision that changes the
syntax or the meaning lands before the merge: a merged RFC describes the
change that is going to be built.

Each RFC gets a shepherd, who asks for the changes it needs and in the
end merges it or turns it away.

An RFC may carry a compatibility clause. A change that is not backwards
compatible in theory may still land when the code in the wild says the
break costs nothing. A claim like that names how it was measured, and
the RFC may need a revision once an implementation attempt says more.

A merged RFC may be edited afterwards to read more clearly. It does not
change meaning that way. A feature built on top of another feature gets
its own RFC.

An RFC closes when there is no agreement that the change is worth
making. A closed RFC is not a closed subject: a pull request may open
again when there is new data, or when the design has moved far enough to
answer what sank it.

Implementation
===

A merged RFC may be built. It carries no date. Some land in days, some
wait months, and a proposal that waits long enough for the ground to
move may be removed so it can be argued again.

When a feature ships, its RFC takes a `**Status**: Implemented` line
above the summary.
