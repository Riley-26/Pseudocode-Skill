# Pseudocode

*Bridging the gap between natural language and code.*

---

Plain English sucks for explaining systems and code. It's a lossy way of carrying structured intent - clunky when describing what to build, and vague when AI is trying to explain what it built. Pseudocode is the bridge between the two, and it's a system that has never been needed more than now. It is precise enough to pin the decisions that matter, but loose enough to skip the parts that don't. No worrying about grammar, just pure intent.

This pseudocode skill is a shared notation that sits between natural language and code. Give your AI this skill, hand it some complicated code, and watch the AI explain the intent behind it in an intuitive way. Less reading, more understanding.

---

## How to read it

The whole system rests on one idea: **the notation tells you how hard to look**. There are three layers:


| Layer  | Looks like             | Read it by                                                                   |
| ------ | ---------------------- | ---------------------------------------------------------------------------- |
| spine  | `FUNC`, `RETURN`       | **skimming** - plumbing, no surprises                                        |
| pinned | `code`                 | **trusting** - exact code, fixed on purpose                                  |
| hole   | `?{...}`               | **scrutinising** - the AI fills this; it's where intent and result can drift |


```pseudocode
FUNC top_priced(products) -> list:
    result <- ?{ products tied for the highest price }
    RETURN result

    MUST:
        - "highest" = the single maximum price; include every product at it
        - ties returned together, ordered by name A-Z
        - empty input returns an empty list, not an error
```

Skim the `FUNC` and `RETURN` lines - they're just structure. Your attention belongs on the hole (`?{ ... }`) and its `MUST` block: that's where the AI makes decisions, and the contract is what holds those decisions to account. Read only that, and you know this returns every product at the maximum price, ties ordered A-Z, with an empty list for empty input - without tracing a single line of real code.

---

## Using it with an AI

It runs in two directions.

**Ask the AI to explain code in pseudocode.** Point it at a file or a function. Instead of paragraphs, you get a skimmable map of intent, with the tricky stuff called out as holes and contracts. Zoom in where you care, skim the rest.

**Write pseudocode to get precise code.** Hand the AI a spec - holes for what you want, contracts for the rules that matter - and it builds the code, then tells you what it filled into each hole and which decisions you didn't pin. You review *intent*, not boilerplate.

```pseudocode
FUNC slugify(title) -> str:
    result <- ?{ url-safe slug of title }
    RETURN result

    MUST:
        - lowercase
        - whitespace and underscores become single hyphens
        - drop characters that aren't a-z, 0-9, or hyphen
        - collapse repeated hyphens; trim the ends
```

Because no language is fixed, that same spec builds into Python, JavaScript, or anything else - only the code changes, the intent doesn't.

**Turning it on and off.** Just ask the AI: "Always pseudocode" makes the AI pair it with everything; "stop" or "no pseudocode" turns it off; by default, it's there when you ask, and it offers itself when it would help.

---

## Writing discipline

The notation is easy; the judgement is the skill. Three habits carry most of it:

- **Holify the uncertain, pin the obvious.** `names[0:5]` is clear - leave it literal. "Rank by relevance" hides a decision - make it a hole. It's backwards, and you bury the clear parts while highlighting the risky ones.
- **Name the failure mode.** "Ties handled correctly" is unreviewable; "ties broken by name, A-Z" is something you can check or test. Write the second kind.
- **Don't out-write the code.** If the pseudocode is as long and dense as what it describes, the code was already its own best explanation - show the code, or zoom out.

---

## Zooming: altitude

You control detail by **altitude**, not by cramming more in. One keyword sets it:

- `@scope map` - the whole file or module. A `FLOW` line for call order, then one line per function with a one-line description and contract headline. For "explain this file".
- `@scope func` - one unit, full holes and contract. Default.

"Show me the file" is `map`; "now zoom into the deduper" is `func`. Below that is the code itself - once you need line-by-line detail you've zoomed past what pseudocode is for, so you read the source. The ladder is **map -> func -> the code**.

```pseudocode
@scope map

FLOW  main -> load -> dedupe -> save -> report

FUNC  load(path) -> rows:
    DESC  read a CSV by header row
    MUST  missing file fails cleanly, not with a crash

FUNC  dedupe(rows) -> rows:
    DESC  collapse to the latest row per user
    MUST  ties broken by first-seen
```

---

## Quick reference

The essentials, the [Legend](references/Legend.md) is the complete canonical list.


| Construct | Looks like                                                  | Means                                          |
| --------- | ----------------------------------------------------------- | ---------------------------------------------- |
| unit      | `FUNC name(args) -> Type:`                                  | a function; the colon opens its body           |
| hole      | `?{ description }`                                          | the AI fills this - scrutinise it              |
| pinned    | `code`                                                      | exact code, kept verbatim                      |
| contract  | `MUST:`                                                     | named behaviours the code must satisfy         |
| assign    | `x <- value`                                                | bind a value to a name                         |
| return    | `RETURN value`                                              | return a value (`RAISE error` for error paths) |
| arrow     | `->`                                                        | "leads to": return types and flow edges (`<-` assigns) |
| altitude  | `@scope map, func`                                          | zoom level                                     |

A trailing colon marks any block with an indented body (`FUNC:`, `IF:`, `MUST:`); single-line keywords like `RETURN` take none. Indent for readability - meaning rests on the keywords, not the columns.

---

## Setup

Two files. `SKILL.md` is the AI-facing half - it teaches the AI how to behave. `references/Legend.md` is the canonical construct list both you and the AI rely on. Keep them nested as `pseudocode-skill/SKILL.md` and `pseudocode-skill/references/Legend.md` so the links resolve, then install it based on your AI's skill mechanism.
