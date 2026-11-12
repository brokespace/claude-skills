---
name: agent-tool-design
description: Use when designing, building, reviewing, or auditing any tool an LLM agent calls (a Kernel tool, a conductor ToolSource, run_cell, finish, spawn), covering its arguments, its name and description, what it returns, and how it errors or refuses. Use before adding or changing a tool's args model, docstring or field descriptions, result model or model_view, or refusals.
---

# Designing tools agents can use

A tool is a prompt the model reads in three places. It reads the
**definition** (name, description, argument schema) on every turn. It has
to write the **call**. It reads the **reply** on its next turn. Design all
three for a capable but literal new hire on their first day. They have
never seen your code and can't ask anyone. They do exactly what the text
says, and they fill in every box you give them, guessing when they don't
know.

The model supplies **intent**; the backend supplies everything else it
can know. Two tests hold that line:

- **The backend test**, for every argument: *could the backend work this
  out itself?* If yes, it is not an argument.
- **The one test**, for every line of output: *would the agent act
  differently if this line were missing?* If not, delete it.

This is one of three skills. `agent-system-design` covers the whole agent
system (prompt, menu, context, termination); `agent-diagnosis` finds what
went wrong in a run. A diagnosis finding at the tool-description,
tool-result or tool-design layer is fixed here.

## How to use this skill

- **New tool:** work steps 1–6 in order. Each ends on a checklist, and the
  step is done when every box is checked.
- **Changing one surface** (arguments, description, result, errors): work
  that step with its reference file, then re-check steps 3 and 6. A changed
  tool needs a true description and a freshly rendered reply.
- **Audit:** score the tool against every checklist below. Each unchecked
  box is a finding. **High**: a failure reported as success, a crash or
  opaque error, silent truncation, output past the size budget, or an
  argument whose natural misuse silently loses or corrupts data. **Med**: a
  miss that costs tokens or rounds. **Low**: polish.

## 1. Scope: one tool per step of the agent's job

Design from the agent's job, not from your endpoints. A tool is a step the
agent thinks in ("add a slide about trust", "find the questions about
price"), not `insert_row` + `reindex` + `refresh_cache`.

- [ ] The tool does one step of the agent's job, in the agent's words.
- [ ] No other tool is always called right before or after it. Tools
      always called in sequence are one tool; a read the write always
      needs happens inside the write.
- [ ] An engineer can say in one sentence when to use it instead of each
      sibling. If they can't, the agent can't either: merge the two or
      sharpen the line between them.
- [ ] One risk level: a read never writes, and a destructive write is its
      own tool.
- [ ] It belongs on this menu. UI-only tools stay off agent menus, and a
      menu stays small (aim for under ~20 tools).

Depth: `references/definitions.md` § Tool set.

## 2. Inputs: intent only

Most bad calls pass the schema check. The values are what's wrong: a
stale revision, an invented id, a field dropped from a resent object.
Every argument the agent supplies is one more place to be wrong, and every
value the backend supplies can't be. Read `references/inputs.md` before
you add or change an argument. It covers each rule below with the failure
it prevents and a worked rewrite.

- [ ] **Nothing the backend can know**: tenant scope, the current, latest,
      or only X, the object this agent is working on, the caller, the
      time, revision counters, dedupe keys, derived fields, values an
      earlier result already holds. Each is filled by the backend, and the
      reply says what was filled.
- [ ] **Partial edits**: an edit sends only what changes. A left-out or
      `null` field keeps its value. Clearing a field is its own explicit
      value. List items are added, removed, or changed one at a time, never
      by resending the whole list.
- [ ] **No concurrency tokens**: the backend tracks what this caller last
      read and catches conflicting writes itself. There is no
      `base_revision` for the agent to carry.
- [ ] **Natural identifiers**: it accepts the code, label, slide number,
      or unique prefix the agent already has, and resolves it. Any handle
      it hands out is short. Positions and offsets are resolved at call
      time and echoed back.
- [ ] **Flat, few, typed**: top-level primitives, enums, and lists of small
      uniform rows. No unions, no field that is required only when another
      field has some value, and no flag pairs that can contradict each
      other. Units go in the names. Internal knobs are left off the schema.
- [ ] **Required means always known**: every required argument is one
      the agent always has at call time. Everything else is optional with
      a safe default. Anything only the user can answer comes back as a
      question to ask them.
- [ ] **Accept and repair**: it coerces types, matches case, prefixes,
      and labels, and reports each repair. It refuses only when the repair
      would be a guess.
- [ ] **Text and code arrive as plain strings**: never JSON encoded inside
      a string argument.
- [ ] **Safe to repeat**: resending the same write doesn't duplicate it,
      and the backend owns the dedupe key. A batch states its rule
      (all-or-nothing, or partial) and its cap.

## 3. Definition: name, description, argument descriptions

The definition is read before every call, whether or not the tool gets
used. It is the biggest single lever on how well a tool is used, and it
also costs tokens on every turn. Every sentence in it must change what the
agent does. Depth, with templates and rewrites:
`references/definitions.md`.

- [ ] The name is a verb and a noun in the agent's words, and at a glance
      it differs from every sibling's name. It uses the menu's verb set.
- [ ] The description says, in order: what it does; when to use it and
      which sibling to use instead; what it needs that isn't obvious; what
      comes back (shape, size, what it fills or repairs); what it changes
      and whether that can be undone or safely repeated; limits that change
      behavior. That usually takes 3–6 sentences.
- [ ] The description never restates the schema, and it has no
      implementation, history, module names, rule codes, or one agent's
      policy.
- [ ] Every argument whose name doesn't fully explain it has a
      description. The description covers what the argument holds (with a
      realistic example value), which tool returns it, what leaving it out
      does, and its units or format.
- [ ] Format-sensitive or complex arguments show 1–5 realistic examples.
- [ ] You read the definition as the model receives it (the rendered
      schema, not the docstring source), and it matches what the tool does
      now.

## 4. Result: high-signal text

The reply is data for the agent's next decision, under a token budget.
Depth, including size budgets, formats, and truncation:
`references/results.md`.

- [ ] Headline first: it says what happened, with a count.
- [ ] Every field passes the one test. There are no internal IDs, hashes,
      timestamps, nulls, defaults, schema tags, or echoed arguments, and
      the full typed model is never serialized.
- [ ] It uses names and labels, not IDs. Any handle is short, and it's
      there only because a later call needs it.
- [ ] Filled or repaired values are stated once.
- [ ] Numbers are rounded to the precision the agent should repeat, with
      units and bases.
- [ ] It's plain text, `key: value` lines, or a small table, with no
      nested JSON dump.
- [ ] Measured size is within budget for the common and large cases
      (typical under ~500 tokens, large under ~2k, never past ~8k), and
      results can be narrowed at the source.
- [ ] Truncation is announced with shown/total and the exact call for
      more. An empty result says what was searched and what to try.
- [ ] A write confirms what changed in one line, plus the handle of
      anything it made. It reports partial batch success row by row.
- [ ] Vocabulary and number style match the sibling tools.
- [ ] If code also reads the reply, code reads the typed result and the
      model reads the text. Neither parses the other's form.

## 5. Errors and refusals: recovery instructions

The model reads the error text and nothing else, so each error is a
complete instruction for what to do next. Depth, including the error
classes and repeat handling: `references/errors.md`.

- [ ] Every failure the agent could fix or route around comes back as an
      error or refusal result, not a crash.
- [ ] Each says what failed, why, and what to do now, naming real tools
      and arguments.
- [ ] All problems are reported in one reply.
- [ ] Each says in words whether to retry.
- [ ] No stack traces, class names, paths, SQL, bare codes, or another
      tenant's data.
- [ ] Ambiguity lists the candidates. A conflict names what changed and
      how to redo the edit. A gap only the user can fill names the
      question to ask them.
- [ ] No failure is dressed up as success, and no no-op is reported as a
      change.
- [ ] A repeat of a failed call gets a sharper reply than the first, and
      repeated failures tell the agent to stop.

## 6. Verify

- [ ] You rendered real replies for the common case, the large case, the
      empty case, and each error class, read them as the agent would, and
      measured their tokens.
- [ ] You rendered the definition as the model gets it and measured its
      tokens. The schema is part of the prompt on every turn.
- [ ] The rendered text is snapshot-tested (not just the typed result), so
      bloat and regressions show in review.
- [ ] Transcripts, when a live run exists, were read for repeated calls,
      ignored results, calls followed by a guess, invented or stale
      argument values, errors followed by the same call, and input-token
      jumps after a tool round. Many invalid-argument errors point at the
      description and its examples; many redundant calls point at result
      size or paging. What the agent leaves out of a call or its reasoning
      is evidence too (`agent-diagnosis` reads runs this way).
- [ ] A changed description or result was checked on a scenario it wasn't
      written around, not only the one that exposed the problem.

## In this repo

- **Scope comes from the bag.** A Kernel tool argument never carries a
  customer or workspace id; `src/tooling/lint/tool_args_vocab.py` fails the
  build.
- **Args models set `extra="forbid"`** (registration fails otherwise). A
  bad call comes back as one refusal listing every problem and the tool's
  arguments (`bq/kernel/boundary/invalid.py`).
- **"The current X" is a resolver default.** `Resolver.default_key` fills a
  left-out id argument, and `missing_next_move` says how to get one when
  it can't (`bq/kernel/resolver.py`). Use it rather than a required id.
- **Writes report `filled` and `repaired`** (see `write_beat`), each with
  what was supplied and why.
- **Reads are tracked for conflicts.** `get_beat` records the caller's read
  (`storyboard/store/field_index.record_read`), and writes check against it.
  An editable object gets the same treatment rather than an agent-facing
  `base_revision`.
- **The description** is the tool's docstring (joined by
  `kernel/groups.describe`). An Expertise `tool_description/<tool>` or
  `tool_description/<menu>/<tool>` entry replaces it for one customer or
  one menu. Argument descriptions are `Field(description=...)` on the args
  model. The model reads `args_model.model_json_schema()` as it stands, so
  every title, default, and enum in it is prompt. `bq tools` prints the
  catalog. `docs/TOOLS.md` is generated, so regenerate it rather than
  editing it.
- **The agent's view is a separate renderer**: `model_view=` on
  `ToolGroup.tool(...)` for a Kernel tool, or `ToolReply.content` for a
  conductor `ToolSource`. Fix bloat there, at the source.
- **Loop code that reads a reply** (a guard, `finished_value`) reads
  `ToolReply.data`, the typed result, through
  `bq.kernel.tool_reply.outcome_of(reply)`. `data` never reaches the model,
  so any tool may take a `model_view`. A guard never parses
  `ToolReply.content`.
- `ToolReply.outcome` is `ok`, `refusal`, or `error`. A refusal's
  `next_move` is the "what to do now" sentence, written as a direct
  instruction that names a real tool or argument.

`references/sources.md` lists where each rule comes from.
