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
| pinned | `` `code ` `` | **trust** - exact target code, fixed by the author |
| hole | `?{...}` | **scrutinise** - natural language fallback. Generative; the model fills it, so this is where what you reviewed and what runs can diverge |

The hole is the risk surface. Reading pseudocode well means spending your attention on the holes and the contract that governs them, and skimming the rest. And that's the idea - it lets a small amount of notation stand in for a large amount of code.

Two rules hold without exception:
- **NEVER invent a construct.** If the Legend lacks one, rephrase with what exists or fold the idea into a hole - never coin a glyph or keyword. Remember - we are translating **intent, not syntax**.
- **NEVER fall back to generic pseudocode.** The value is in the shared convention; made-up notation defeats the point.

---
## 0. MODES

Three states. The default is conservative; the user moves it up or down in plain language.

- **Default.** When pseudocode is invoked, use it. The first time it comes up in a session, note *once* that it's available for related work - e.g. "I can lay the rest of this out in pseudocode too, if useful" - then drop it. Only repeat the offer for non-trivial tasks, but never force it, and never gate an answer behind a plan the user didn't ask for.
- **off** - set by "no pseudocode", "stop", "turn it off". Suppress everything, including the one-time note, until the user reactivates it. Once off, stay off; don't re-offer in later replies.
- **always** - set by "always pseudocode", "pseudocode everything". Proactively pair non-trivial code with pseudocode for the rest of the session - shown just before the code as a preview, or alongside it as an explanation. Either way the pseudocode previews or describes the code; it is never the source the code is compiled from. Still skip trivial code, and don't gate: deliver both together unless the user explicitly asks to approve the plan first.

---
## 1. THE FRAME

Every pseudocode block is the same four parts, in this order: **headers, signature, body, contract**. Only two things inside are yours to decide - *what each hole produces* and *what the contract guarantees*. Everything else is fixed. A block is not free-form; it is this frame, filled in.

That makes the frame the one shape you always produce and always expect. Match it exactly: a line that isn't part of the frame is an error, not a variation. Every later section either specifies one part of this frame or teaches how to fill one of the two slots - the shape itself never changes.

```pseudocode
@scope func

KIND name(args) -> Type:
    name <- ?{ ... }
    name <- call(...)
    RETURN value

    MUST:
        - guarantee
        - guarantee
```

Filled in:

```pseudocode
@scope func
@goal  quote a basket price, applying an optional discount

FUNC price_quote(items, discount?) -> Money:
    subtotal <- ?{ sum the prices of the items }
    total    <- apply_discount(subtotal, discount)
    RETURN total

    MUST:
        - out-of-stock items are excluded from subtotal
        - discount defaults to none (no reduction)
        - total never goes negative; a discount over 100% clamps to zero
```

One glance gives the whole shape: an optional header, a signature (kind, arguments, `-> Type`, colon), a body of bindings - one hole for the part that needs judgement, one plain call for the part that's delegated - closed by `RETURN`, and a `MUST` of guarantees. There is no control flow in the body, and there never is: a loop or a branch is something a hole produces, not a line you write.

**Fixed - identical in every block:**

- the signature: a kind keyword, arguments, `-> Type`, a colon
- the body: bindings only - `name <- ?{ ... }` or `name <- call(...)` - and nothing else
- the terminal: the body ends in `RETURN` or `RAISE`
- the `MUST:` block

**Judgement - the two slots you fill:**

- the inside of each `?{ ... }` - what it produces
- the assertions under `MUST` - what must be true of the result

The next section specifies each part; *Scope* covers how this frame flattens at `map`; *Judgement* is where the two slots get filled well; and the pre-flight check validates a block against this frame before it's emitted.

---
## 2. THE PARTS

Four specs, one per frame part. Each is a hard rule, not a guideline - the form is fixed here so the two judgement slots are the *only* place anything is decided.

### Headers

Zero or more `@` directives, one per line, at the top of the block, no colon. Use only these:

- `@target <lang>` - the target language, written as its file format (`py`, `js`, ...). **Omit** it for language-agnostic pseudocode.
- `@goal <text>` - one line stating what the unit achieves (intent, not mechanism).
- `@scope map | func` - the altitude. Default is `func`; state `map` for a whole-file view.
- `@import <list>` - modules the unit needs.

### Signature

```pseudocode
KIND name(args) -> Type:
```

- `KIND` is one of `FUNC`, `METHOD`, `CLASS`, `ASYNC`, `ROUTE`. The set is closed; never invent a kind. Choose the one that names what the unit *is* - dictated by the code, not chosen freely.
- `args` are names, comma-separated. Mark an optional argument with a trailing `?` (`client?`). Types are optional but recommended: `name: Type`.
- `-> Type` is the return type; a unit that returns nothing uses `-> none`.
- The trailing **colon** ends the signature and opens the body.

### Body

The body is a sequence of **bindings**, ending in one terminal. Nothing else appears in a body.

A **binding** is `name <- <expr>`, where `<expr>` is exactly one of:

- a **hole** - `?{ ... }`
- a **call** to another unit - `derive_rows(world, registry)`
- a **pinned literal** - `exact target code` (for exact code, including arithmetic like `subtotal * (1 - rate)`)
- a **reference** - a variable, a field (`obj.field`), or a slice (`rows[0:5]`)

The **terminal** is `RETURN value`. A body ends in exactly one.

Two rules make the body strict, and they are not optional:

- **No control flow.** There is no `IF`, `FOR`, `WHILE`, `TRY`, `CATCH`, `CONTINUE`, or `BREAK` - none of these exist. A loop or a branch is not a body line; it is something a hole *produces*. If you are about to write one, you are transcribing - fold it into a hole instead.
- **One exit.** A body has a single `RETURN`. Error paths do not branch off it: "raises `X` when `Y`" is a `MUST` guarantee, never a body line. `RAISE` is a terminal only when rejecting is the unit's entire job - a pure guard.

#### The hole

The hole `?{ ... }` is judgement slot 1 - the one generative part of a block, and the reason a contract exists. Its **form** is fixed: a `?{ }` on the right side of a binding, holding a natural-language description of what to produce, with no nested notation and one hole per binding. Its **content** - what to put in it, and when a part should be a hole rather than a pin or a call - is judgement, covered in *Judgement*.

### Contract

```pseudocode
MUST:
    - <guarantee>
    - <guarantee>
```

One `MUST:` block per unit, at the end of the body. Each line is a **guarantee**: something that must be true of the result, that a test could pass or fail.

A `MUST` line is never:

- a **configuration value** - a model name, a token budget, a default. That is data; it goes in a binding or a hole.
- an **implementation step** - *how* the result is computed. That is the mechanism; it goes in the hole.

What belongs here: error paths, edge cases, ordering, what counts as a match, invariants. *What* to guarantee is judgement slot 2 (see *Judgement*); that each line must *be* a guarantee is fixed here.

---
## 3. SCOPE

One `@scope` keyword sets the altitude. There are two, and below them is the code itself:

- `func` (default) - one unit, the full frame: signature, body of bindings, contract. Every block so far has been `func`.
- `map` - a whole file or module. The frame flattens: no bodies, no holes, one line of summary per unit.
- **the code** - below `func`, a line of pseudocode would only restate a line of code, so you stop and read the source. The ladder is `map` -> `func` -> **the code**, and it matches how reading goes: "explain this file" is `map`, "zoom into this function" is `func`, "what does this line do" is the code.

### The map projection

At `map`, a block collapses to its outline. A `FLOW` line gives the call or data order across the file:

```pseudocode
@scope map

FLOW load -> dedupe -> save
```

Then **one entry per unit** - its signature, a one-line `DESC`, and a one-line `MUST` headline. No body, no holes:

```pseudocode
FUNC load(path) -> rows:
    DESC read a CSV by header row
    MUST missing file fails cleanly, not with a crash

FUNC dedupe(rows) -> rows:
    DESC  collapse to the latest row per user
    MUST  ties broken by first-seen

FUNC  save(rows, path) -> none:
    DESC  write rows back as CSV
    MUST  never leave a partially written file on failure
```

The rules are strict:

- `DESC` is **one line** - what the unit does, as intent.
- `MUST` is **one line** - the single most important guarantee, no colon, no list.
- **More than one guarantee is not allowed at `map`.** If a unit needs a fuller contract, that is the signal to zoom it to `func`, where the full `MUST:` block lives - never to stack lines at `map`.
- No bodies, no holes, no terminals. `map` is the shape of the file, not its contents.

Zooming a unit from `map` to `func` expands its one-line `DESC`/`MUST` back into the full frame; zooming below `func` lands on the code.

---
## 4. JUDGEMENT

The frame fixed everything except two slots. This is where they get filled well. It reads less like rules than the sections above, because it *is* judgement - but every call here resolves to a test: *could this line pass or fail?* If you can't answer that, the line isn't done.

### Filling holes

One principle governs the slot: **holify the uncertain, pin the obvious.** A real decision - anything a reader would need to scrutinise - is a hole. Exact code you already know is a pinned literal. A named unit is a call. The skill is putting each part in the right one. Two ways that go wrong, each with a fix.

**Mechanism holifies.** Setup, payload construction, defaulting, retries - none of it is the intent, so it collapses into the hole, *even though each line is a legal binding*. Legal is not the bar; meaningful is.

Over-spelled - every mechanical step its own binding:

```pseudocode
payload <- ?{ build the notify JSON from message and user }
headers <- ?{ auth headers from api_key }
sent    <- post(`BASE + "/notify"`, payload, headers)
```

Holified - the mechanism folds into one hole:

```pseudocode
sent <- ?{ POST the message to the notify endpoint, authenticated with api_key }
```

**Steps or rules?** Decide what the function *is*. When its value is its rules, the spine collapses to produce-and-return and the contract carries its weight:

```pseudocode
FUNC parse_port(raw) -> int:
    port <- ?{ read raw as an integer }
    RETURN port

    MUST:
        - surrounding whitespace is ignored — " 80 " reads as 80
        - non-numeric input, or a value outside 1..65535, returns 3000
```

The parse is trivial; the rules - trim, range, default - are the whole point, so they live in the contract. The opposite case is a genuine algorithm (a parser, a diff, a layout pass): there the steps *are* the intent, so they stay as bindings. Ask which kind you have before writing the body.

**The transcription test.** About one line of pseudocode per line of code means you are transcribing, not abstracting. Collapse mechanism into holes until the spine is just the shape. If the block is as long and dense as the code, the code was already its own best description - show the code.

### Writing guarantees

The form is fixed in *parts*: a `MUST` line is a guarantee, never config or a step. The judgement here is making each one **sharp**.

**Name the failure mode.** A guarantee that can't fail a test guarantees nothing.

Weak - nothing a test could check:

```pseudocode
MUST:
    - input is validated
```

Sharp - each line is a claim a test could pass or fail:

```pseudocode
MUST:
    - empty input RAISEs ValueError, never returns []
    - duplicate keys keep the last occurrence
```

Cover the edges that carry risk - empty, error, ordering, duplicates, what counts as a match - and stop there. A contract that restates the obvious is as useless as one that's vague; every line should earn its place by ruling out a way the result could be wrong.

---
## 5. A WORKED EXAMPLE

Judgement is easiest to see on the one function that fights it hardest: an async model call wrapped in a retry loop with validation and error handling. Real code first, then the block.

```python
async def extract_invoice(raw_text, client=None, tax_table=None):
    client = client or default_client()
    tax_table = tax_table or load_tax_table()
    system = build_system_prompt(EXTRACTION_RULES)
    messages = [{"role": "user", "content": wrap_with_guard(raw_text)}]

    for _ in range(2):
        text, _ = await call_model(
            client, system, messages,
            max_tokens=2000, model="claude-opus-4-8",
        )
        try:
            data = parse_json(text)
            data.pop("invoice_id", None)
            data.pop("line_total", None)
            data = clamp_quantities(data)
            invoice = Invoice.model_validate(data)
            invoice.line_total = compute_line_total(invoice, tax_table)
            return invoice
        except (JSONDecodeError, ValidationError) as exc:
            messages += [
                {"role": "assistant", "content": text},
                {"role": "user", "content": f"Invalid: {exc}. Return only corrected JSON."},
            ]
            continue

    raise ExtractionError("invoice validation failed twice")
```

Almost none of that is the intent. The loop, the try/except, the continue, the message-appending are one idea - get a validated invoice from the model, repairing once if it comes back malformed. The block says that and stops:

```pseudocode
@target py
@goal  extract a validated Invoice from raw text via the model; repair once on failure

ASYNC extract_invoice(raw_text, client?, tax_table?) -> Invoice:
    system   <- ?{ system prompt carrying the extraction rules + return-only-JSON }
    messages <- ?{ initial user turn: raw_text wrapped in an injection guard }
    invoice  <- ?{ call the model, then parse and validate the reply into an Invoice,
                   retrying once with the error fed back on failure }
    invoice.line_total <- compute_line_total(invoice, tax_table)
    RETURN invoice

    MUST:
        - invalid JSON or a schema violation triggers exactly one corrective retry;
          a second failure RAISEs ExtractionError, never a partial Invoice
        - model-supplied invoice_id and line_total are stripped before validation
        - quantities outside 1–999 clamp to 1, never rejected
        - line_total is computed after a successful parse, never copied from the model
```

What each move is, and why:

- **The loop and the try/except became one hole.** "Retry once with the error fed back" is a rule about the work, not a step in it - so it leaves the body. The hole names the *outcome* (a validated Invoice); spelling "strip fences -> parse -> pop -> validate -> on error append and retry" would be transcription that merely moved inside the braces.
- **The default-argument lines vanished.** `client or default_client()` is plumbing the `client?` mark already implies. Writing it into the body is the exact mechanism a hole absorbs or omission drops - here, dropped, because nothing downstream needs to watch it happen.
- **The rules moved to `MUST` as guarantees.** Strip these fields, clamp that range, derive after parse, retry-then-raise - each is a claim a test could fail, not an instruction. That is what makes the hole above safe to trust at a glance.
- **What's not in the contract.** The model id, `max_tokens=2000` - configuration, not guarantees. It stays inside the hole's call; a `MUST` line asserting "Opus is used" would be config wearing a contract's clothes.

The body went from eighteen lines to four, and the four that remain are the only four that carry a decision. That ratio is the test: if a transform doesn't collapse like this, the holes are still holding mechanism.

---
## 6. DIRECTIONS

The notation runs two ways. In both, the code is the ground truth - the pseudocode describes it, never replaces it.

**Explanation - code you have, intent you want.** Read the code and render it as a block at the right altitude: spine for shape, holes for the parts that carry real logic, a contract for the behaviour the code states only implicitly. Nothing is disclosed here, because there is nothing to keep honest - the pseudocode *is* the account of the code.

**Instruction - intent you have, code you want.** Write the block as a spec; the model builds from it. Here the pseudocode is the source, so the build is followed by a **disclosure loop** - two short reports, not a verdict:

- *what filled each hole* - one line per hole, saying what was produced.
- *what wasn't pinned* - any decision the contract didn't fix, surfaced for a check ("dropped null-priced items; confirm that's intended").

The disclosure never certifies the code is correct - a model can't reliably grade its own output. It reports what it did and points at what to test. That is honest; a green check it can't back is not.

**The AI can open the contract.** Instead of waiting for a written spec, the model may draft the block first for the user to read or edit. Once the user engages with it, it is an agreed contract and follows the same build-then-disclose path. What the model never does is treat its own undisclosed draft as the source and compile from it silently - that puts an unreviewed spec on the critical path.

---
## 7. WHEN NOT TO PSEUDOCODE

Pseudocode is a cost - notation to learn and read. Skip it when the cost outruns the benefit:

- **Trivial code.** A one-liner or an obvious getter has no decision to scrutinise and no hole to write.
- **When it would run as long as the code.** If the block is as long and dense as what it describes, the code is already its own clearest account - show the code.
- **When prose is clearer.** For a single simple point, a sentence beats a block. Reach for the notation when there is structure or a contract worth pinning.
- **When the user wants the answer, not a lesson.** If code is asked for directly and the mode isn't `always`, give the code; offer the pseudocode once if it would help, never impose it.

The test: does the pseudocode let a reader skip something they would otherwise have to trace? If not, it is overhead.

---
## 8. GUARDRAILS

Before any block is shown, it passes this check - the backstop that holds every model, light or strong, to the frame. A block that fails a line is defective, not a variation; fix it before emitting.

1. **Serialisation** - emit the whole block inside a single ```pseudocode fence; its keywords are literal, never rendered as markdown - `MUST` is not a bullet list, `DESC` is not a heading.
2. **Frame** - exactly optional headers, a `KIND ... :` signature, a body, a `MUST:`. Nothing sits outside the frame.
3. **Body** - every line is a binding (`name <- ...`) or the single `RETURN`. No `IF`, `FOR`, `WHILE`, `TRY`, `CATCH`, `CONTINUE`, or `BREAK`; fold any into a hole.
4. **Constructs** - every keyword and glyph is in the Legend. Nothing invented, no improvised notation.
5. **Contract** - each `MUST` line is a guarantee a test could fail, not a config value or a step. At `map`, one line per unit.
6. **Density** - the block is shorter than the code it describes; about one line per line means collapse mechanism into holes.

One rule sits outside the checklist because it's about generation, not shape: when you build code from pseudocode, never certify the result - disclose what was done and point at what to test. A model can't back a guarantee it only asserts.