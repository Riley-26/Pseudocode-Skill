---
name: pseudocode-skill
description: Explains, plans, and verifies code using our proprietary *pseudocode* — an intent-based bridge between AI-generated code and natural language. Use this skill whenever the user mentions "pseudocode" explicitly, OR asks to "explain", "break down", "walk through", or "plan" code, OR asks what code does / how it works / why it works. Also use it proactively before or after generating non-trivial code when the pseudocode setting is active. Conventions are defined in Legend.md; do NOT fall back to generic pseudocode.
---

# Pseudocode Skill

Act as a strict semantic translator between our *pseudocode* system and target-language code. The pseudocode you provide is ALWAYS from our proprietary conventions, defined in the [legend](Legend.md). You ARE explaining your code's *intent* with *pseudocode*, NOT planning the task with ad-hoc notation. This system is intent-based: you infer and express **intent**, not syntax.

NEVER generate generic pseudocode. If you don't have a construct, rephrase with the legend — don't invent one.

---
## 0. SETTING — three modes

The user chooses how proactive the skill is. Detect the mode from what they say; when unsure, default to **offer**.

| Mode | The skill... | Set by phrases like |
|------|--------------|---------------------|
| **auto** | emits a *pseudocode* plan **before every** code generation and asks the user to confirm before writing code | "always pseudocode", "pseudocode everything", "plan before you code" |
| **offer** *(default)* | generates code, then **offers** to explain it with *pseudocode* — does not force it | "back to normal", "just offer it" |
| **off** | is disabled until reactivated; you may only *offer* to explain with *pseudocode*, nothing more | "turn off pseudocode", "no pseudocode", "stop explaining" |

A block-level `@mode` header overrides the session mode for that one block. When the skill is **off**, respect it — do not inject pseudocode or repeated offers into every reply.

---
## 1. INFER OR CREATE

You will either **read** existing pseudocode/intent and infer the code, or **write** *pseudocode* to express a goal. Refer to the [legend](Legend.md) for conventions you MUST follow.

### 1.1 Reading
1. Using the user's input and surrounding context, understand what they want to achieve, and infer their target language.
2. They may write in our *pseudocode*, generic pseudocode, or plain natural language — translate all of it into our conventions before reasoning about the code.

### 1.2 Writing
1. In **auto** mode, provide a *pseudocode* plan before generating code, and confirm it looks right before continuing.
2. For **large** outputs, generate the code, then offer a *pseudocode* explanation afterward — don't front-load it.
3. Provide *pseudocode* on request for planning or teaching — it isn't limited to pre-coding.

---
## 2. THE GOAL

*Pseudocode* bridges complicated AI-generated code and natural language. As models generate code faster and more accurately, pure reliance on them leads to:

- **Discouraged users** — code arrives faster than understanding can keep up; pausing to understand feels like it kills momentum.
- **Rampant token use** — a model brute-forcing a bug is a dog chasing its tail.
- **A doomed project** — building on code nobody understands mortgages the future.

The goal is to **align the user with the AI** so both know exactly what the code does at all times, **while keeping development fast**. It does this by:

1. **Intuitive explanation** — combining familiar programming structure with natural-language intent, so meaning is translated, not syntax.
2. **Lower time and token spend** — a conventional shared notation short-circuits brute-force debugging loops.
3. **A consistent, language-agnostic ecosystem** — generic pseudocode has no rules; ours is fixed and referable, so any programmer can pick it up.

The payoff only lands if the *pseudocode* — specifically its `MUST:` contract (§4) — becomes the thing the user reviews **instead of** reading every line of generated code. §7 is what makes that trustworthy.

---
## 3. EXAMPLES

The same source intent, expressed three ways. Note how little of each block is mechanical spine and how much carries intent the code can't self-document.

### 3.1 Python — first 5 active users from a CSV

```pseudocode
@target py
@goal   return the first 5 active users from a CSV

FUNC top_users(path: str) -> list[str]
	rows  <- ?{ read CSV at `path`, using the header row }
	names <- ?{ name of each row where status == "active" }
	RETURN names[0:5]

	MUST:
		- rows accessed by header name, not column position
		- an inactive row contributes nothing
		- returns <= 5 names, kept in file order

MAIN:
	EMIT top_users(`sys.argv[1]`)
```

### 3.2 JavaScript — top 3 priciest in-stock products

Different content, same shape — and the contract pins an ordering decision that the code would otherwise leave to chance.

```pseudocode
@target js
@goal   return the 3 most expensive in-stock products

FUNC top_priced(products) -> list
	stocked <- ?{ products where inStock is true }
	ranked  <- ?{ stocked sorted by price, most expensive first }
	RETURN ranked[0:3]

	MUST:
		- price compared as a number, never as a string
		- ties in price broken by name, A->Z (deterministic order)
		- out-of-stock products excluded before ranking
		- returns <= 3, even if fewer are in stock

MAIN:
	EMIT top_priced(`require('./catalog.json')`)
```

### 3.3 Language-agnostic — dedupe, preserving first-seen order

No `@target`, no pinned literals. The contract carries the one decision that matters: *which* order survives.

```pseudocode
@goal   remove duplicates from a list, keeping first appearance

FUNC dedupe(items) -> list
	seen   <- ?{ track which values have appeared }
	result <- ?{ each item, only the first time it is seen }
	RETURN result

	MUST:
		- order follows first appearance, NOT sorted order
		- the second+ occurrence of a value is dropped, not the first
		- original input is not mutated
```

---
## 4. HOW TO WRITE A CONTRACT

The `MUST:` block is the deliverable. Everything else is scaffolding; this is what lets the user stop reading the body. Write it well or the system gives nothing.

**Holify the uncertain, pin the obvious.** A hole `?{ }` belongs where code would be *guessed* or where mechanics don't matter. Code that is already clear — a slice, a return — stays literal. Restating obvious code as a hole only loses precision.

**Every hole needs a contract line.** A hole alone is non-deterministic; a hole plus a `MUST:` line is "fill it however you like, but it must satisfy this." That pairing is the whole trick.

**Name the failure mode — do not imply it.** Capable models infer intent from terse lines; cheaper models need the failure spelled out. Write the line as the mistake you're forbidding, concretely:

- weak: `- handle status correctly`
- strong: `- match status exactly; do NOT lowercase or trim whitespace`

**Make silent accidents into stated decisions.** If the original code's behaviour was merely *what the idiom happened to do* — file order, case sensitivity, tie-breaking — write it down. Writing it forces you to *decide* it, which is the point.

**Adversarial case — when the idiom fights the contract.** The contract earns its keep where a model's default instinct pulls the *wrong* way. A model's reflex is often to normalise case (`status.toLowerCase() === "active"`). If your data uses exact-match statuses, that reflex is a bug:

```pseudocode
FUNC active_only(rows) -> list
	result <- ?{ rows where status is active }
	RETURN result

	MUST:
		- status matched EXACTLY: "active" only
		- do NOT lowercase, uppercase, or trim before comparing
		- "Active", "ACTIVE", " active " do NOT count
```

If the contract binds, the model overrides its idiom. If you can't tell from reading the output whether it did, the contract isn't specific enough yet — tighten it.

---
## 5. WHEN NOT TO PSEUDOCODE

Even when the skill is active, skip *pseudocode* when it adds nothing or gets in the way:

- **Trivial / self-evident code** — one-liners, boilerplate, a getter. A hole would just restate obvious code.
- **Code-only flow** — the user is mid-task and has asked for code directly, or is debugging under time pressure. Don't gate their answer behind a plan.
- **Non-code tasks** — prose, config, data files with no logic or intent to carry.
- **When it would be longer than what it explains** without adding any intent the reader couldn't already see.

In these cases, in **auto** mode, write the code and (briefly) offer pseudocode rather than forcing a plan. Never let the skill slow the user down on work that doesn't need it.

---
## 6. BLACKLIST

Hard rules. These prevent the skill from becoming annoying, misleading, or wrong:

- **NEVER** generate generic pseudocode — only the conventions in [Legend.md](Legend.md).
- **NEVER** invent a glyph or keyword that isn't in the legend. Rephrase instead.
- **NEVER** holify self-evident code, or restate control flow the reader can already see.
- **NEVER** write a vague `MUST:` line. Each must name a concrete, checkable failure mode.
- **NEVER** claim a contract held without actually running the §7 conformance check against the generated code.
- **NEVER** use the skill to stall, refuse, or pad a response. It explains code; it does not gate it.
- **NEVER** force a confirm-prompt or a plan onto trivial snippets, or onto a user who asked for code only.
- **NEVER** keep injecting pseudocode or offers once the user has set the mode to **off**.

---
## 7. CONFORMANCE CHECK

This is what makes reviewing the *pseudocode* a real substitute for reading the body. After generating or receiving code that has a contract, verify it — line by line.

For each `MUST:` line, do two things:

1. **Verdict** — does the generated code satisfy it? `PASS` / `FAIL`, with the specific line or construct that decides it.
2. **Silent additions** — note anything the code did that the contract did **not** ask for (error handling, case normalisation, extra sorting, pretty-printing). These aren't necessarily wrong, but they're decisions the model made on its own — surface them so the user can rule on them rather than discover them later.

Report it compactly, e.g.:

```
CONTRACT CHECK — top_users
  [PASS] header-name access      -> uses DictReader
  [PASS] inactive contributes 0  -> filtered before append
  [PASS] <= 5, file order        -> [:5] over file-order iteration
  NOTE  added newline='' and a FileNotFoundError guard — not specified; OK?
```

If any line is `FAIL`, fix the code, not the contract — unless the contract itself was wrong, in which case fix it and say so. A `FAIL` you can see beats a silent bug you can't.