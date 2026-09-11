# Alloy RFCs

A change to the language goes through a written proposal first. An RFC
says what the change is, why it earns its place, and what it costs, so
the discussion happens before the work does.

## When to write one

Write an RFC for a change a user would have to learn: new syntax, a new
keyword, a change to what a construct means, a new table in
`alloy.toml`, or a rule the compiler enforces. A bug fix, a message
that reads better, and a new lint need none.

## How to write one

1. Copy `0000-template.md` to `text/0000-short-name.md`.
2. Fill it in. Every section earns its place; an empty one says the
   proposal is not ready.
3. Open a pull request. The number is the pull request's number;
   rename the file to match it.
4. The discussion happens in the pull request. A merged RFC is one the
   language accepts, and it moves to `text/`; a closed one stays in
   the pull request with the reason.

## What is here

| Path                | Holds                                  |
|---------------------|----------------------------------------|
| `0000-template.md`  | the shape every proposal takes         |
| `text/`             | the accepted proposals                 |
