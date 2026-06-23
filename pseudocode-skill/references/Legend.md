# Pseudocode Skill — Legend

The authoritative reference for *pseudocode* conventions. Every construct compiles to a
predictable target-language equivalent. If a construct you need isn't here, do **not** invent
one - rephrase using what exists, or carry the idea inside a hole `?{ }`.

This is the construct reference; SKILL.md is the operating spec. The two share one shape - *headers, signature, body, contract* - and this file defines each piece of it.

## Three layers, three trust levels

*Pseudocode* is read by eye, and the notation tells the reader how hard to look:

| Layer        | Looks like        | Meaning                                   | How to read it      |
|--------------|-------------------|-------------------------------------------|---------------------|
| spine        | `FUNC`, `RETURN`  | mechanical structure the model handles    | skim                |
| pinned       | `` `code` ``      | exact target code, fixed by the author    | trust as-is         |
| hole         | `?{ ... }`        | generative - the model fills it           | **scrutinise most** |

A hole is the **risk surface**: the one place where what you reviewed and what actually runs
can diverge. Every hole should be governed by a `MUST:` block (see Contract).

## Readability

The idea is to translate readability through structure and appropriate spacing.

- **Indentation.** Use indentation to promote readability. It is not exactly essential, but it is highly beneficial. Users are familiar with indentation, it structures ideas consistently.
- **Colons.** Adopt a trailing colon at the end of a keyword block that opens to an indented block`FUNC:`, `CLASS:`, `MAIN:` (though at `@scope map`, `MUST` is a one-line headline, it doesn't need the colon) etc. Single-line keywords take no colon: `RETURN`, `RAISE` etc.
- **Spacing.** For areas that have multiple lines of the same importance, you could line them up with multiple spaces. E.g. `@target py` one whitespace, lines up with `@goal  return the ...` with two whitespaces. Not required, may improve readability.
- **Arrows.** `->` reads "leads to": it types a return (`-> rows`) and links `FLOW` edges (`load -> save`) alike. `<-` is assignment, pointing at the receiver (`x <- value`). The notation uses no other arrows.

## Headers

| Token            | Meaning                                                            |
|------------------|-------------------------------------------------------------------|
| `@target <lang>` | target language in its file format (`py`, `js`, ...). **Omit entirely** for language-agnostic pseudocode. |
| `@goal <text>`   | one line: what this unit achieves (intent, not mechanics).        |
| `@import <list>` | optional - modules/packages the unit needs.                       |
| `@scope <scope>` | altitude - map (file/module) or func (one function, default). See SKILL.md, 3. Altitude. |

Example uses:

```pseudocode
@target py
@goal   return the first 5 active users from a CSV
@import csv, sys
@scope  func

...
```

## Signature & structural kinds

```
FUNC name(arg: Type, opt?: Type) -> Type
```

A unit begins with a **structural-kind keyword** naming what it is, so the reader knows how to read it. Types are optional but recommended - they pin the interface. Language-agnostic blocks may drop them. For multiple return types, use a paren tuple `-> (rows, errors)`, and for structured returns, prefer `-> dict` but define the structure within the `MUST` ("result has keys id, name, total"). Smaller structured returns like `-> {id: str, total: int}` are acceptable. A unit that returns nothing is written `-> none`. An argument the caller may omit is marked with a trailing `?` (`client?`); the default the body supplies is plumbing — holify or omit it, never spell it out in the spine (`client <- client OR default` is exactly the kind of mechanism that belongs inside a hole).

Both return types and `FLOW` edges use `->` (see Map scope).

The kind set is **closed**, not extensible (an open list just becomes generic pseudocode). A new keyword earns a slot only if it changes *the contract you would write*:

| Keyword | Unit | Notes |
| --- | --- | --- |
| `FUNC` | plain function | baseline |
| `METHOD` | function owned by a class | nested under its `CLASS`, indented; drops the dotted prefix |
| `CLASS` | stateful unit | has state + invariants `MUST` covers them |
| `ASYNC` | async function | await / ordering / partial-failure are real contract terms |
| `ROUTE` | HTTP endpoint | `ROUTE POST /users -> 201 - the verb is an argument` |

Admission test for any future keyword: *does it change the contract?* `PATCH` vs `PUT` does not (it's a `DESC`/`MUST` detail), so it stays an argument, not a keyword.

## Body

Between the signature's colon and the terminal, the body is a sequence of **bindings** closed by **exactly one terminal** - nothing else. No bare statements, no control flow.

A **binding** is `name <- <expr>`, and `<expr>` is exactly one of four things:

| `<expr>` | Looks like | Defined in |
| --- | --- | --- |
| hole | `?{ ... }` | Holes |
| pinned literal | `` `code` `` | Pinned literals |
| call | `derive(world, registry)` | a call to another unit |
| reference | `x`, `obj.field`, `rows[0:5]` | Access & slicing |

The arrow points **at the receiver**. There is no append or mutate operator - build a collection inside a hole and assign the whole result (`names <- ?{ ... }`).

**No control flow.** Loops, branches, resource handling, and error handling have no constructs - no `FOR`, `IF`, `WHILE`, `OPEN`, `TRY`, `CATCH`, `CONTINUE`, or `BREAK`. A loop or branch is mechanism: what it *achieves* goes in a hole, and the rules that govern it (how many times, which branch wins, what gets raised) go in a `MUST`. A control-flow construct is an invitation to transcribe the code instead of abstracting it.

**One exit.** A body ends in a single terminal. An error path doesn't branch off it - "raises `X` when `Y`" is a `MUST` guarantee, not a body line (see Terminals).

## Holes

```
?{ natural-language description of what to produce }
```

A generative hole - the model fills it at generation time. One hole per binding, no nested notation inside it. Use a hole where code would only be *guessed* or where the mechanics don't matter - **never** to restate code that is already clear. `names[0:5]` stays literal; "filter rows where status is active" becomes a hole.

## Pinned literals

```
`target code`
```

Exact target-language code, reproduced verbatim and trusted. Use for the handful of details you want fixed precisely (`` `sys.argv[1]` ``, `` `subtotal * (1 - rate)` ``). Language-specific by definition — a language-agnostic block should avoid pinned literals.

## Terminals

| Token                 | Compiles to                                  |
|-----------------------|----------------------------------------------|
| `RETURN value`        | return                                       |
| `RAISE error`         | raise / throw - an error path is a **named** outcome, not a hidden one |

A terminal names a unit's outcome - what is hands back - and a body has exactly **one** (see *One exit*, above). `RETURN` is the usual terminal. `RAISE` is a terminal only when rejecting is the unit's whole job - a pure guard; a conditional raise ("raises `X` when `Y`") is not a terminal but a `MUST` guarantee. Everything between the signature and the terminal is a binding or the contract.

## Map scope - `FLOW` and `DESC`

Used only at `@scope map`, where the unit of intent is the function/class, not the statement.

```
FLOW load -> dedupe -> save
```

`FLOW` states the call/data order between units - the big picture that a per-function view can't carry. Branch it when the file isn't a straight pipeline: `save -> {ok: report, fail: rollback}`.

```
FUNC load(path) -> rows
    DESC read CSV by header
    MUST header required
```

At map scope each unit is a kind+signature line with indented `DESC` (one line) and `MUST` (one line, no colon). No bodies, no holes. Blank line between top-level units; a `CLASS` and its `METHOD`s stay grouped and indented.

**One guarantee per unit at `map`.** The `MUST` headline carries the single most important guarantee. If a unit needs more, that is the signal to zoom in to `@scope func`, where the full `MUST:` block lives - never stack `MUST` lines at map. See SKILL.md, *Scope*.

## Contract — the deliverable

```
MUST:
    - <assertion>
    - <assertion>
```

One contract per `FUNC`, placed at the end of its body. Each line is a **guarantee**: something that must be true of the result, that a test could pass or fail - the things code is otherwise silent about (access style, ordering, what counts as a match, edge-case handling). This sharpening technique is to *name the failure mode* rather than assert a vague property.

A `MUST` line is **never** a configuration value (a model name, a budget, a default - that is data, and goes in a binding or a hole) and **never** an implementation step (*how* the result is computed - that is mechanism and goes in the hole). This block is the load-bearing part of the whole system; see SKILL.md, *The parts* for its form and *Judgement* for how to write one well.

At `@scope map` the same `MUST` keyword condenses to a single inline *headline* per unit; zooming that unit to `@scope func` expands the headline back into the full block above.

## Access & slicing (language-agnostic)

| Token        | Meaning                                              |
|--------------|------------------------------------------------------|
| `obj.field`  | field access by name                                 |
| `value[0:5]` | slice — compiles to `[:5]` / `.slice(0, 5)` / equivalent |

These are **reference** expressions - one of the four things a binding's right side can be (see Body).