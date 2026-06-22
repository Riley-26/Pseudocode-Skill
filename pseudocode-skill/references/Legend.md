# Pseudocode Skill — Legend

The authoritative reference for *pseudocode* conventions. Every construct compiles to a
predictable target-language equivalent. If a construct you need isn't here, do **not** invent
one - rephrase using what exists, or carry the idea inside a hole `?{ }`.

## Three layers, three trust levels

*Pseudocode* is read by eye, and the notation tells the reader how hard to look:

| Layer        | Looks like        | Meaning                                   | How to read it      |
|--------------|-------------------|-------------------------------------------|---------------------|
| spine        | `FUNC`, `RETURN`  | mechanical structure the model handles    | skim                |
| pinned       | `` `code` ``      | exact target code, fixed by the author    | trust as-is         |
| hole         | `?{ ... }`        | generative - the model fills it           | **scrutinise most** |

A hole is the **risk surface**: the one place where what you reviewed and what actually runs
can diverge. Every hole should be governed by a `MUST:` line (see Contract).

## Readability

The idea is to translate readability through structure and appropriate spacing.

- **Indentation** - Use indentation to promote readability. It is not exactly essential, but it is highly beneficial. Users are familiar with indentation, it structures ideas consistently.
- **Colons** - Adopt a trailing colon at the end of a keyword block that opens to an indented block. Required.
- **Spacing** - For areas that have multiple lines of the same importance, you could line them up with multiple spaces. E.g. `@target py` one whitespace, lines up with `@goal  return the ...` with two whitespaces. Not required, may improve readability.

## Headers

| Token            | Meaning                                                            |
|------------------|-------------------------------------------------------------------|
| `@target <lang>` | target language in its file format (`py`, `js`, ...). **Omit entirely** for language-agnostic pseudocode. |
| `@goal <text>`   | one line: what this unit achieves (intent, not mechanics).        |
| `@import <list>` | optional - modules/packages the unit needs.                       |
| `@scope <scope>` | altitude - map (file/module), func (one function, default), or trace (line-level). See SKILL.md, 3. Altitude. |

```pseudocode
@target py
@goal   return the first 5 active users from a CSV
@import csv, sys
@scope  func
```

## Signature

```
FUNC name(arg: Type) -> Type
```

A unit begins with a **structural-kind keyword** naming what it is, so the reader knows how to read it. Types are optional but recommended - they pin the interface. Language-agnostic blocks may drop them. For multiple return types, use a paren tuple `-> (rows, errors)`, and for structured returns, prefer `-> dict` but define the structure within the `MUST` ("result has keys id, name, total"). Smaller structured returns like `-> {id: str, total: int}` are acceptable. A unit that returns nothing is written `-> none`.

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

## Storage

```
x <- value
value -> x
```

Assign / bind. The arrow points **at the receiver**. There is no separate append operator - build a collection inside a hole and assign the whole result (`names <- ?{ ... }`).

## Holes

```
?{ natural-language description of what to produce }
```

A generative hole. The model fills it at generation time. Use a hole where code would only be *guessed* or where the mechanics don't matter - **never** to restate code that is already clear. `names[0:5]` stays literal; "filter rows where status is active" becomes a hole.

## Pinned literals

```
`target code`
```

Exact target-language code, reproduced verbatim and trusted. Use for the handful of details
you want fixed precisely (`` `sys.argv[1]` ``, an exact flag, a specific call). Language-specific
by definition — a language-agnostic block should avoid pinned literals.

## Control spine

| Token                 | Compiles to                                  |
|-----------------------|----------------------------------------------|
| `FOR item IN source`  | iteration                                    |
| `IF condition`        | branch / guard                               |
| `OPEN path AS handle` | resource binding (`with` / `using` / `try-with-resources`) |
| `RETURN value`        | return                                       |
| `RAISE error`         | raise                                        |

Use spine keywords **only when the structure itself is the intent.** If a single hole can
carry the loop more clearly (as in the dedup example under Directions), prefer the hole.

## Map scope - `FLOW` and `DESC`

Used only at `@scope map`, where the unit of intent is the function/class, not the statement.

```
FLOW main -> load -> dedupe -> save -> report
```

`FLOW` states the call/data order between units - the big picture that a per-function view can't carry. Branch it when the file isn't a straight pipeline: `save -> {ok: report, fail: rollback}`.

```
FUNC load(path) -> rows
    DESC read CSV by header
    MUST header required
```

At map scope each unit is a kind+signature line with indented `DESC` (one line) and `MUST` (one line). No bodies, no holes. If a `MUST` needs multiple lines, that would be a cue to zoom to the func scope. Blank line between top-level units; a `CLASS` and its `METHOD`s stay grouped and indented. See SKILL.md, 3. Altitude.

## Contract — the deliverable

```
MUST:
    - <assertion>
    - <assertion>
```

One contract per `FUNC`, placed at the end of its body. Each line is a **named failure mode** the generated code must satisfy - the things code is otherwise silent about (access style, ordering, what counts as a match, edge-case handling). This block holds up the whole system; see SKILL.md, 1. Holes and their Contracts, for how to write one and how to check it held.

At `@scope map` the same `MUST` keyword condenses to a single inline *headline* per unit; zooming that unit to `@scope func` expands the headline back into the full block above.

## Entry & output

| Token        | Compiles to                                              |
|--------------|---------------------------------------------------------|
| `MAIN:`      | entry point (`if __name__ == "__main__"` / `require.main === module` / `main()`) |
| `EMIT value` | output (`print` / `console.log` / stdout)               |

## Access & slicing (language-agnostic)

| Token        | Meaning                                              |
|--------------|------------------------------------------------------|
| `obj.field`  | field access by name                                 |
| `value[0:5]` | slice — compiles to `[:5]` / `.slice(0, 5)` / equivalent |