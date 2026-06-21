# Pseudocode Skill — Legend

The authoritative reference for *pseudocode* conventions. Every construct compiles to a
predictable target-language equivalent. If a construct you need isn't here, do **not** invent
one — rephrase using what exists, or carry the idea inside a hole `?{ }`.

## Three layers, three trust levels

*Pseudocode* is read by eye, and the notation tells the reader how hard to look:

| Layer        | Looks like        | Meaning                                   | How to read it      |
|--------------|-------------------|-------------------------------------------|---------------------|
| spine        | `FUNC`, `RETURN`  | mechanical structure the model handles    | skim                |
| pinned       | `` `code` ``      | exact target code, fixed by the author    | trust as-is         |
| hole         | `?{ ... }`        | generative — the model fills it           | **scrutinise most** |

A hole is the **risk surface**: the one place where what you reviewed and what actually runs
can diverge. Every hole should be governed by a `MUST:` line (see Contract).

## Headers

| Token            | Meaning                                                            |
|------------------|-------------------------------------------------------------------|
| `@target <lang>` | target language (`py`, `js`, ...). **Omit entirely** for language-agnostic pseudocode. |
| `@goal <text>`   | one line: what this unit achieves (intent, not mechanics).        |
| `@import <list>` | optional — modules/packages the unit needs.                       |
| `@mode <mode>`   | optional — overrides the session setting for this block (`auto` \| `offer` \| `off`). See SKILL.md §0. |

## Signature

```
FUNC name(arg: Type) -> Type
```

Types are optional but recommended — they pin the interface so the reader knows the shape
without reading the body. Language-agnostic blocks may drop the types.

## Storage

```
x <- value
```

Assign / bind. The arrow points **at the receiver**. There is no separate append operator —
build a collection inside a hole and assign the whole result (`names <- ?{ ... }`).

## Holes

```
?{ natural-language description of what to produce }
```

A generative hole. The model fills it at generation time. Use a hole where code would only
be *guessed* or where the mechanics don't matter — **never** to restate code that is already
clear. `names[0:5]` stays literal; "filter rows where status is active" becomes a hole.

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

Use spine keywords **only when the structure itself is the intent.** If a single hole can
carry the loop more clearly (as in the CSV example), prefer the hole.

## Contract — the deliverable

```
MUST:
    - <assertion>
    - <assertion>
```

One contract per `FUNC`, placed at the end of its body. Each line is a **named failure mode**
the generated code must satisfy — the things code is otherwise silent about (access style,
ordering, what counts as a match, edge-case handling). This block is the load-bearing part of
the whole system; see SKILL.md §4 for how to write one, and §7 for how to check it held.

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