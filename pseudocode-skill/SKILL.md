---
name: pseudocode
description: A shared, conventional pseudocode notation that bridges code and natural language - both the user and the AI read and write it, primarily to explain or review existing code, and also to generate code from a pseudocode spec. Use whenever pseudocode is explicitly invoked - when the user asks to explain, review, or walk through code in pseudocode; to plan, express, or draft code as pseudocode; provides pseudocode to generate code from; or refers to the notation by name. Always use it in these cases, but do NOT trigger on a bare request to explain or write code that makes no mention of pseudocode. Conventions are defined in Legend.md; never fall back to generic pseudocode.
---

# Pseudocode Skill

Pseudocode is a shared, fixed notation that carries the *intent* of code between a user and AI - both read it and write it. It expresses **intent, not syntax** and it is not made-up shorthand: every construct comes from the [Legend](references/Legend.md). The code is always the ground truth; the pseudocode is a lens on it, never a substitute — it describes the code, it doesn't replace reading it.

Its defining idea: the notation tells you **how hard to look** at each part. Three layers -

| Layer | Looks like | How to read it |
| --- | --- | --- |
| spine | `FUNC`, `RETURN`, `IF` | **skim** - mechanical structure, no surprises |
| pinned | `code` | **trust** - exact target code, fixed by the author |
| hole | `?{...}` | **scrutinise** - natural language fallback. Generative; the model fills it, so this is where what you reviewed and what runs can diverge |

The hole is the risk surface. Reading pseudocode well means spending your attention on the holes and the contract that governs them, and skimming the rest. And that's the idea - it lets a small amount of notation stand in for a large amount of code.

Two rules hold without exception:
- **Never invent a construct.** If the Legend lacks one, rephrase with what exists or fold the idea into a hole - never coin a glyph or keyword. Remember - we are translating **intent, not syntax**.
- **Never fall back to generic pseudocode.** The value is in the shared convention; made-up notation defeats the point.

---
## 0. MODES

Three states. The default is conservative; the user moves it up or down in plain language.

- **Default.** When pseudocode is invoked, use it. The first time it comes up in a session, note *once* that it's available for related work - e.g. "I can lay the rest of this out in pseudocode too, if useful" - then drop it. Only repeat the offer for non-trivial tasks, but never force it, and never gate an answer behind a plan the user didn't ask for.
- **off** - set by "no pseudocode", "stop", "turn it off". Suppress everything, including the one-time note, until the user reactivates it. Once off, stay off; don't re-offer in later replies.
- **always** - set by "always pseudocode", "pseudocode everything". Proactively pair non-trivial code with pseudocode for the rest of the session - shown just before the code as a preview, or alongside it as an explanation. Either way the pseudocode previews or describes the code; it is never the source the code is compiled from. Still skip trivial code, and don't gate: deliver both together unless the user explicitly asks to approve the plan first.

---
## 1. HOLES AND THEIR CONTRACTS

A hole `?{...}` is the one generative part of the notation - the model fills it, so it's where what you reviewed can drift apart from what runs. A **contract** closes that gap: a `MUST:` block that pins the hole's behaviour in terms you can check. The discipline is *specificity*. A vague contract is unreviewable:

```pseudocode
FUNC top_priced(products) -> list:
    result <- ?{ products with the highest price }
    RETURN result

    MUST:
        - ties handled correctly
```

"Handled correctly" can't be confirmed against the code - it never says what *correct* means. Name the behaviour instead:

```pseudocode
FUNC top_priced(products) -> list:
    result <- ?{ products tied for the highest price }
    RETURN result

    MUST:
        - "highest" = the single maximum price; include EVERY product at it
        - ties returned together, ordered by name A-Z
        - empty input returns an empty list, not an error
```

Now each line is a behaviour you can point to in the code or turn into a test. The contract earns its keep most where it **fights the model's default reflex**. A model's instinct is to normalise case (`status.lower() == "active"`); if your data uses exact statuses, that instinct is a bug:

```pseudocode
FUNC active_only(rows) -> list:
    result <- ?{ rows whose status is active }
    RETURN result

    MUST:
        - compare status EXACTLY: "active" only
        - do NOT lowercase, uppercase, or trim first
        - "Active", "ACTIVE", " active " do NOT count
```

If you can't tell from the result whether the contract bound, it isn't specific enough yet - tighten it. 

**Closing the loop: check the code against the contract.** After the code is written, do two things - and note they are not the same kind of claim.

*Disclose* what filled each hole, and flag anything done that the contract didn't ask for. This is the reliable half - it only reports what was done:

- `top_priced` - took the max price, returned all matches sorted by name. As specified.
- `top_priced` - skipped products with a null price; the contract didn't mention them. Confirm you want them dropped, not treated as 0.
- `active_only` - exact `==`, no normalisation. As specified.

*Advise, don't certify*. A model grading its own semantic claims can be confidently wrong. Point instead at the holes that carry real risk:

`active_only` relies on exact-match statuses - if the data might contain `"Active"` or trailing spaces, test that path before trusting it.

A pointer to what to test is something the model can stand behind. A green checkmark it can't verify is not.

## 2. DIRECTIONS & EXAMPLES

The notation runs both ways, and in both the code stays the ground truth.

**Explanation - you have the code, you want the intent.** Read the code and render its intent as pseudocode: skimming spine for structure, a hole for the part that carries the real logic, and a contract naming the behaviour the code is otherwise silent about. Collapse mechanical loops into the hole - the reader wants the *decision*, not the transcription.

```python
def latest_per_user(events):
    seen = {}
    for e in events:
        uid = e["user_id"]
        if uid not in seen or e["timestamp"] > seen[uid]["timestamp"]:
            seen[uid] = e
    return list(seen.values())
```

becomes

```pseudocode
FUNC latest_per_user(events) -> list:
    result <- ?{ keep only the latest event per user }
    RETURN result

    MUST:
        - group by event.user_id
        - within a user, keep the greatest event.timestamp
        - on equal timestamps, keep the first one encountered
```

The loop vanishes; what's left is the rule a reader actually needs - including the tie behaviour (first-seen wins) that the raw code states only implicitly through `>`.

**Instruction - you have the intent, you want the code.** Write the pseudocode as a spec; the model builds from it and runs the disclosure loop. Here the pseudocode *is* the source, so the loop isn't optional - it's what keeps an unverified source honest.

```pseudocode
FUNC slugify(title) -> str:
    result <- ?{ url-safe slug of the title }
    RETURN result

    MUST:
        - lowercase
        - whitespace and underscores become single hyphens
        - drop characters that aren't a-z, 0-9, or hyphen
        - collapse repeated hyphens; trim leading and trailing ones
```

Because no target language is fixed, the same spec builds into either - only the code differs:

```python
def slugify(title):
    s = re.sub(r"[\s_]+", "-", title.lower())
    s = re.sub(r"[^a-z0-9-]", "", s)
    return re.sub(r"-+", "-", s).strip("-")
```

```js
function slugify(title) {
    let s = title.toLowerCase().replace(/[\s_]+/g, "-");
    s = s.replace(/[^a-z0-9-]/g, "");
    return s.replace(/-+/g, "-").replace(/^-+|-+$/g, "");
}
```

Then close the loop:

- `slugify` - lowercased, separators to single hyphens, dropped invalid characters, collapsed and trimmed. As specified.
- Order: invalid characters are dropped *after* separators convert, so `"a_b"` becomes `"a-b"`. The contract didn't pin that order - flag if you wanted it reversed.
- Empty or all-symbol titles return `""`. Confirm that's acceptable, or pin it.

**The AI can open the contract, too.** Rather than wait for a hand-written spec, the model may draft pseudocode first for the user to read or edit; the moment the user engages with it, it's an agreed contract and follows the same build-then-disclose path. What the model must never do is treat its *own* unreviewed pseudocode as the source and compile from it silently - that puts the pseudocode back on the critical path with no one having checked it.

---
## 3. ALTITUDE

When code gets large, the wrong fix is to cram more into one view. The right fix is to **change altitude** - zoom out for shape, in for detail. One `@scope` keyword sets it:

- `map` - the whole file or module. A `FLOW` line for call/data order, then one kind+signature line per unit with a one-line `DESC` and one-line `MUST`. No bodies, no holes. For "explain this file".
- `func` - one unit, full holes and contract. The default, and the altitude of every example so far.
- `trace` - line by line, dropping below the hole when a single step needs scrutiny. For "walk me through this exact computation".

This is what lets the notation scale: you don't write *denser* pseudocode for bigger code, you write the same density at a higher altitude. It also matches how explanation actually goes - "show me the file" is `map`, "now zoom into dedupe" is `func`, "what's that one line doing" is `trace`.

`map` - shape over detail (a unit that returns nothing is written `-> none`)

```pseudocode
@scope map

FLOW main -> load -> dedupe -> save -> report

FUNC load(path) -> rows:
    DESC read a CSV by header row
    MUST missing file fails cleanly, not with a crash

FUNC dedupe(rows) -> rows:
    DESC collapse to the latest row per user
    MUST ties broken by first-seen

FUNC save(rows, path) -> none:
    DESC write rows back as CSV
    MUST never leave a partially written file on failure

FUNC report(kept, dropped) -> none:
    DESC print a one-line summary
```

`trace` - the same `slugify` from above, zoomed in below its single hole:

```pseudocode
@scope trace

FUNC slugify(title) -> str:
    s <- `title.lower()`
    s <- ?{ replace each run of whitespace or underscore with one hyphen }
    s <- ?{ drop characters outside a-z, 0-9, hyphen }
    s <- ?{ collapse repeated hyphens, then trim both ends }
    RETURN s
```

Same function, finer altitude: the one hole becomes four steps, each scrutinised on its own. Zoom this far only when a step earns it - `trace` everywhere is just code with extra syntax.

---
## 4. WHEN NOT TO PSEUDOCODE

Pseudocode is a cost - notation to learn and read. Skip it when the cost outruns the benefit.

- **Trivial code.** A one-line helper or an obvious getter needs no contract. If there's no decision to scrutinise, there's no hole to write.
- **When the pseudocode would run as long as the code.** If translating adds lines without removing uncertainty, the code was already its own clearest description - show the code.
- **When prose is clearer.** "Sorts users by signup date, newest first" beats a notation block for a simple, single-purpose function. Reach for the notation when there's structure or a contract worth pinning.
- **When the user wants the answer, not the lesson.** If someone asks for code directly and isn't in `always` mode, give them the code; offer pseudocode once if it'd help, don't impose it.

The test: does the pseudocode let the reader skip something they'd otherwise have to trace? If not, it's overhead.

## 5. BLACKLIST

The two cardinal rules:

- **Never invent a construct.** Every glyph and keyword comes from the Legend. If it lacks one, rephrase with what exists or carry the idea inside a hole - an open keyword set rots straight back into generic pseudocode.
- **Never fall back to generic, improvised pseudocode.** The value is the *shared* convention; improvised notation throws away the only thing that makes it a language.

And the quieter mistakes:

- **Don't holify the obvious or pin the uncertain.** `names[0:5]` stays literal; "rank by relevance" is a hole. It's backwards, and you hide the certain parts while highlighting the risky ones.
- **Don't let notation get as dense as the code.** A hole that only restates the line it replaces earns nothing - if a block is as hard to read as the source, raise the altitude instead.
- **Don't certify what you can only describe.** No PASS/FAIL verdicts on a hole's semantics - disclose what was done and point at what to test.
- **Don't compile from your own unreviewed pseudocode.** A model's self-authored spec is a proposal until the user engages with it; until then, code comes from normal reasoning, not from treating the draft as a verified source.

