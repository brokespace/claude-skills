# Results: high-signal text

A tool's output is a prompt. Every result is text the model reads on its
next turn, under a token budget, with no one to ask.

## Anatomy of a result

A result reads top-down in this order, and stops as soon as it's done:

1. **Headline**: one line saying what happened, in plain words, with the
   count. `Found 3 questions about brand trust (of 212 in the survey).`
2. **The data**: only what feeds the agent's next decision.
3. **Scope note**, only if the result is partial: what was left out and
   the exact call that gets it.
4. **Next step**, only if it isn't obvious: the exact tool and argument.

Put the answer first. Models attend most to the start and end of a block,
and a headline lets the agent stop reading.

## What goes in

Include:
- Fields the agent will **quote, compare, decide on, or pass to another
  tool**.
- **Names, labels, and text instead of IDs.** Models handle
  `Q12 "How much do you trust…"` far better than `a3f9c2e1-…`. Keep a
  stable handle only when a later call needs it, and keep it short (`Q12`,
  `ex-4`, a slug).
- **Values the tool filled or repaired**, stated once:
  `Using wave 2024-Q3 (the latest; none was given).`
- **Scope and freshness** when they change what's true: population, base
  size, filter applied, as-of wave.
- **Numbers at the precision the agent should repeat**, with units and
  bases: `42% (n=1,204)`, not `0.4198712`. The model copies what it sees.

Exclude:
- Internal IDs, UUIDs, hashes, row versions, revision counters, and tenant
  IDs.
- Timestamps, audit fields, `created_by`, and storage paths, unless the
  task is about them.
- `null`s, empty lists, `false` flags, and fields at their default.
  Leaving a field out already says "nothing here".
- An echo of the arguments the agent just sent (it knows them), except a
  value the tool changed or filled.
- The same fact twice (a label plus a code plus a description that says
  the same thing), and derived fields the agent can see at a glance.
- Schema metadata, type tags, discriminators, and model class names.
- Whole objects when the agent asked for one fact. **Never serialize the
  full Pydantic model as the agent's view.** Build a view for the model.

## Format

- **Default to plain text with light structure**: a headline, then
  `key: value` lines or a short list. It's close to what models are trained
  on, it costs the fewest tokens, and it needs no escaping.
- **Uniform rows go in a small markdown table**, which gave the best mix of
  tokens and accuracy. **Per-record lookups go in `key: value` blocks**,
  which were the most accurate format in benchmarks. Avoid CSV and
  pipe-delimited dumps for anything the model has to look things up in.
- **No nested JSON for the model to dig through.** Flatten. If you must
  return JSON (because a caller parses it in code), keep it flat, use short
  meaningful keys, drop nulls, and serialize keys in a stable order. An
  unstable key order breaks the prompt cache.
- **Use one vocabulary across every tool.** The same thing gets the same
  name, label style, and number format everywhere. If one tool says "ask",
  another mustn't say "question_ref". An argument and the result field
  that feeds it share a name.
- **Output is data, not orders.** Tool output supplies facts and hints. It
  never overrides the system prompt. Text that came from users or
  customers (survey verbatims, uploaded files) is quoted as data, not
  written as instructions.

## Size

- **Budget every tool.** House defaults: a typical result is **under ~500
  tokens**, a large one **under ~2k**, and nothing goes past **~8k**.
  Anything past the ceiling is an error that says how to narrow the call,
  not a dump. Measure the tokens of real outputs rather than guessing.
- **Filter at the source.** Give the tool the arguments to narrow its
  results (search terms, a limit, a single field or section), and pick
  small defaults. A `list_everything` tool is a design bug; a `search` or a
  `get_one` is the fix.
- **Always announce truncation.** Say how many were shown, the total, and
  the exact call for more:
  `Showing 20 of 212. Narrow with query=…, or call again with offset=20.`
  Silent truncation makes agents report partial data as complete.
- **Large artifacts get a handle and a summary**, not the content: say what
  the artifact holds, how big it is, and the call that reads a slice of it.
- **The summary is the default.** Offer detail on request (a `detail` flag,
  or a separate get tool) rather than always sending the detailed view.

## Empty results

Never return a bare empty list. Say what was searched and what to try:
`No questions match "loyalty" in this survey. Closest topics: retention (Q30–Q34), NPS (Q41). Try one of those, or search by question code.`

An empty result the agent misreads becomes a hallucinated answer.

## Action tools (writes)

- Confirm in one line what changed, plus the handle of anything made:
  `Added slide 4 "Trust by region" to the storyboard (now 7 slides).`
- Report anything filled, repaired, or coerced, so the agent's picture
  stays true: `Title shortened to 60 characters.`
- Don't echo the whole object back. Return the new state only when the
  agent needs it for its next edit.
- In a batch, **report partial success**: what succeeded, what failed and
  why, and the call that retries only the failures.

## When code reads the reply too

A loop guard, a recipe, or a finish check may read the same reply the
model reads. Those are two readers with two needs. Code gets the typed
result, and the model gets the rendered text. Never shape the model's text
so a guard can parse it, and never make code parse text written for the
model. When the transport carries only one of them, the fix is to carry
both, not to give the model the typed dump.

## Worked rewrites

These are illustrative shapes, not copies of real tool output.

**A lookup that returns whole objects.** Before: every matching question,
fully serialized (one call ran to 148 KB):

```json
{"items": [{"id": "5b1e0c9a-7d2f-4f0e-9a51-0c3de1f0b2aa", "dataset_id": "d-91f…",
  "code": "Q12", "type": "single", "label": "Trust in brand",
  "text": "How much do you trust each of the following brands?",
  "options": [{"value": "1", "label": "Not at all", "order": 0, "is_other": false,
  "exclusive": false, "meta": null}, …], "grid": {"rows": [...], "cols": [...]},
  "created_at": "2026-09-12T10:31:02Z", "revision": 7, "filters": null, …}, …]}
```

After:

```
3 questions match "trust" (of 212).
Q12 Trust in brand — grid, 8 brands × 5-point scale (Not at all … Completely)
Q13 Trust vs last year — single, 3 options (Less, Same, More)
Q40 Trust in advertising — 5-point scale
Call graph.question("Q12") for one question's full options.
```

What changed: there's a headline with the count, and the code and label
replace the UUID. The option set is summarized as its range. The ids,
timestamps, revision, nulls, and flags are gone, and the detail moved to a
get-one call.

**A write that echoes the world.** Before: the whole 40-field storyboard
object, after adding one slide. After:

```
Added slide 4 "Trust by region" (now 7 slides). Title shortened to 60 characters.
```

**Silent truncation.** Before: the first 50 rows, with nothing to say
there were 212. After:

```
Showing 50 of 212 respondents' verbatims for Q30, most recent first.
Narrow with query="price", or call again with offset=50.
```

**Empty.** Before: `{"items": []}`. After:

```
No questions match "loyalty". Closest topics: retention (Q30–Q34), NPS (Q41).
```
