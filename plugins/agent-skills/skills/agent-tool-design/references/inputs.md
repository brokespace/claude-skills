# Inputs: the model sends intent

Tool calls mostly fail on values, not on shape. In one audit of seven
frontier models, only 2.1% of failed calls broke the schema. The rest were
schema-valid with a wrong value. The hardest values to get right are deeply
nested fields, fields required only when another field has some value, and
values copied from an earlier call's result. Each argument the agent must
supply is another chance to get one wrong. Each value the backend supplies
can't be wrong that way.

Designers who are agents themselves reach for the same defaults: mirror
the database row, require the whole object, add a revision counter "for
safety", make every id required. Each of those pushes the backend's work
onto the model. The sections below are the replacements.

## 1. The backend test: what never becomes an argument

| The agent is asked for | The backend already has it from | Instead |
|---|---|---|
| Tenant, customer, workspace | the bag | never an argument (lint-enforced here) |
| "The current / latest / only" X | a query | optional; default to it and say so |
| The object this agent is working on | the task, lane, or principal | default to it |
| Who is calling, when | the principal, the clock | fill it |
| A revision, version, or etag to "prove" freshness | its own read tracking | §3 |
| A dedupe or idempotency key | the run, turn, and call ids | §9 |
| A derived field (a count, a total, a slug, `cut_sources`) | the other fields | compute it; keep it off the schema |
| Values from an earlier result (six numbers, a label, a base) | that result | accept the handle (`ev-3`), not a copy of its values |

When the backend fills something, the reply says so once:
`Rethemed the published deck (v7; none was named) as Co-op.`

The description of an optional argument says what leaving it out does:
`Default: the current published version.`

Two findings from this repo's audit had exactly this shape.
`deck.retheme` required a `version_id`, although "show this as Co-op"
always means the published deck. `deck.build_deck` required a
`storyboard_assessment_id` that the agent had to copy from another tool,
although the newest completed release assessment is one query away.

## 2. Partial edits

**The failure:** changing one field requires sending the whole object.
The model has to reproduce every other field exactly, and it doesn't. It
copies stale values from an older read, leaves out fields it never saw,
"improves" fields it wasn't asked to touch, and elides the rest ("…other
fields unchanged"). Models get lazy when they have to rewrite a whole
thing to change part of it; aider measured this on whole-file edits. The
result is silent data loss, reported as a success. And every write now
needs a read first, just to have the object to resend.

**The rules:**

- **Send only what changes.** Every field the agent leaves out keeps its
  value. Nested maps merge: `fields: {"caption": "…"}` changes the caption
  and nothing else.
- **Treat a left-out field and `null` the same way: leave it as is.**
  Strict modes for function calling (OpenAI's `strict: true`, for one)
  make every property required and nullable. A strict-mode model
  therefore sends `null` for every field it isn't changing. If `null`
  meant "clear", every strict-mode edit would wipe the object. Make
  clearing explicit: an empty string for text, an empty list, or a
  `clear: ["subtitle"]` argument.
- **Edit list items one at a time.** Take `add_*` / `remove_*` / an
  update keyed by the item's name or handle. Or accept a new list only if
  the tool diffs it against the stored one and reports what it added and
  removed. "Resend every item" is the failure above in list form. This
  repo's `study.edit_design` replaces `entities` whole, so an agent editing
  one entity has to resend every other entity exactly.
- **Edit text by anchor.** Take the old snippet and its replacement, where
  the snippet must match exactly once. Don't take a rewrite of the whole
  text or character offsets. Anthropic tried several edit strategies for
  its coding agent, and string replacement was the most reliable. A snippet
  that matches zero times or more than once is a refusal that says which.
- **Say what changed.** `Changed caption on slide 4.` A field sent with its
  current value is reported as unchanged. A patch that changes nothing is
  a refusal naming the fields it could have changed, not an `ok`
  (`update_section` returned `ok` for an empty patch).

## 3. Concurrency without tokens

**The failure:** every write requires a `base_revision`, `expected_version`,
or etag. The agent has to read the object first, carry the number across
calls (copying a value from an earlier result is the hardest kind of
argument to get right), and send it back correctly. It gets this wrong in
every direction. It copies a stale number from an earlier read, increments
it itself ("edit 6 after edit 5"), invents one, or retries a conflict with
the same number. Meanwhile the backend knows every object's revision and
can count edits itself.

**The rules:**

- **The backend tracks reads.** Record what each caller last read (per
  object, or per field) and check each write against it. The agent just
  reads, then writes. Claude Code's file editing works this way: `Edit`
  refuses a file that hasn't been read, or that has changed since it was
  read. In this repo `get_beat` records the read, and writes detect a field
  someone else changed since then.
- **Merge when it's safe.** If the fields changed since the read don't
  overlap the fields being written, apply the write and say so:
  `Changed caption. (The headline was edited by someone else since your
  read; it is now "Trust fell in the North".)`
- **On a real conflict, refuse with the current state:** what changed,
  its current value, and the one call that fixes it.
  `Slide 4's caption changed since you read it (now "…"). Redo your edit on
  that text; your change was not applied.` Put the current values in the
  refusal so the redo doesn't need another read.
- **A fresh read always clears the conflict.** A check that keeps failing
  after a re-read wedges the agent. Claude Code's issue tracker holds bug
  reports of exactly that ("File has been unexpectedly modified" after a
  re-read).
- **Anchored edits check themselves.** An old snippet that must still
  match needs no token: if the text changed, the snippet doesn't match.
- **A human UI can still send a revision.** That's the HTTP route's args
  model, not the agent's. The agent-facing tool doesn't expose it
  (`patch_beat` carries `base_revision` and is on no agent menu).

## 4. Identifiers

- **Accept what the agent has in words**: a code (`Q12`), a label
  ("Overall satisfaction"), a slide number, a name, a unique prefix. The
  backend resolves it and says so when the match wasn't exact:
  `Using Q7 "Overall satisfaction" (matched "overall sat").`
- **Hand out short handles.** When the agent must hold an id, make it
  short and scoped (`Q12`, `ex-4`, `s3`, a slug). A 30-character ULID
  costs about 12 tokens every time it's copied, and it's easy to mistype.
- **Avoid positions.** Indexes, offsets, line numbers, and "the third item"
  shift after every insert, and models handle them badly (aider: GPT is
  "terrible at working with source code line numbers"). Use a stable
  handle or a content anchor. When a position is how the user talks
  ("slide 3"), accept it, resolve it at call time, and echo what it
  resolved to: `slide 3 = "Trust by region"`.
- **One name per kind of id, everywhere.** Use `beat_id` in every tool,
  never `beat` in one, `id` in another, and `beat_ref` in a third. Write
  `user_id`, not `user`. An id one tool hands out is accepted, in the same
  form, by every tool that takes that kind of id.

## 5. Shape: flat, few, typed

- **Keep it flat.** Use top-level strings, numbers, booleans, and enums,
  plus lists of small uniform rows (`beats: [{headline, section}]`). Keep
  object nesting to one level at most. Deep nesting is where models place
  the right value at the wrong depth or invent keys.
- **No unions of object shapes** (`anyOf` / `oneOf`). Split them into
  separate tools, or use one `kind` enum plus flat optional fields.
  `qc.add_finding_comment`'s `TextAnchor | SlideAnchor` union made its
  schema about 495 tokens, and the agent had nothing to fill it with.
- **No conditional requirements** ("`next_hop_id` is required when
  `next_hop_type` is `router`"). A rule like that lives only in prose, and
  models miss it. Split the tool, or make the field optional with a default
  and refuse a wrong combination with the exact fix.
- **No flags that can contradict each other.** `toggle_light(on, off)`
  admits `on=true, off=true` (OpenAI's example). Use one enum of the modes
  (`mode: "preview" | "apply"`, not `dry_run` + `confirm`), or separate
  tools.
- **Keep it to a few arguments.** Every optional argument is a decision the
  agent makes on every call, and a token cost on every turn. Leave internal
  knobs (seeds, replicate counts, batch sizes, cache switches) at their
  defaults and off the schema unless the agent's job needs them.
- **Types say what the value is.** Numbers are numbers and booleans are
  booleans. Dates are ISO 8601, with the timezone rule stated. Units go in
  the name: `timeout_seconds`, `share_pct` (0–100) rather than `share`.
- **Use enums for closed sets**, with values in the agent's words (`"bar"`,
  not `"BAR_V2"`). For a large or open set, take a string and resolve it
  with a did-you-mean: a 300-value enum costs its full length in the
  prompt on every turn.
- **Build the agent's own args model.** It can be smaller than the HTTP
  route's. A field only the UI can fill stays off the agent's schema.

## 6. Required and optional

- **Required means the agent always knows the value at call time**, from
  the user's words or an earlier result.
- **Models fill required boxes they can't answer.** They rarely stop to ask
  (ShortcutsBench found every model tested "exceptionally poor" at noticing
  it needed input). A required argument the agent can't know invites an
  invented value.
- **An optional argument has a safe default.** The argument's description
  states the default, and the reply says when it was used.
- **When only the user knows**, the tool refuses with the question to ask:
  `"Co-opp" is not a published brand. Published: Co-op, Acme. Ask the user
  which one they mean.`

## 7. Accept and repair

Be liberal in what you accept. aider's lesson from edit formats was to be
"maximally flexible when interpreting" what the model sent.

- **Coerce** `"5"` → `5`, `"true"` → `true`, a single value where a list is
  expected → a one-item list, and case, whitespace, and a stray trailing
  period.
- **Resolve** a label sent where a code was expected, a unique prefix, or a
  close match above a confidence threshold.
- **Report every repair** in the reply, once: `Title shortened to 60
  characters.`
- **Refuse only when the repair would be a guess.** Then list the
  candidates (`errors.md` § What an error says).

## 8. Text, code, and structured payloads

- **Long text and code get their own plain string argument.** Models write
  worse code when it's wrapped in JSON. In aider's benchmark every model
  scored lower and made more syntax errors, and strict JSON mode didn't
  help.
- **Never send JSON encoded inside a string argument**
  (`"spec": "{\"axes\": …}"`). Double escaping is where models break. A
  structured payload is the schema; code is the single argument of a code
  tool (`run_cell`).
- **Prose content is Markdown or plain text**, not HTML or a custom
  mini-language the agent has to learn from the description.

## 9. Batches, retries, idempotency

- **Take a list** when agents naturally do N of the same thing (`add_beat`
  takes rows). State the cap.
- **Pick one batch rule and state it in the description.** Either
  all-or-nothing: check every row first, report every bad row, apply none.
  Or partial: apply the good rows, report each bad one, and give the call
  that retries only those.
- **Resending a write is safe.** The same create sent twice returns what the
  first one made (`Already added: slide 4 "Trust by region"`), not a
  duplicate.
- **The backend owns the dedupe key.** It's the natural key of the content,
  or the run, turn, and call ids, never a value the model makes up.
- **Long work returns a handle straight away**, plus a way to wait on it or
  read it, so the agent never has to invent a polling loop.

## 10. Worked rewrites

These are illustrative shapes, not copies of real tool output.

**Edit one field**

Before: the whole beat, plus a counter the agent had to fetch first.
```json
{"beat": {"beat_id": "bea_01M3T3WMYHJX343QRB1BMH1GFA", "section_id": "sec_…",
  "shape": "chart", "headline": "…", "fields": {"caption": "New caption",
  "alt": "…", "fit": "contain", …}, "exhibit": {…}, "pins": […], …},
 "base_revision": 5}
```
After:
```json
{"beat_id": "s4", "fields": {"caption": "New caption"}}
```
```
Changed caption on slide 4 "Trust by region". Nothing else changed.
```
What changed: the agent sends one field and a short handle. The backend
merges, knows the revision, and checks for conflicts against the agent's
own last read.

**An id the backend can default**

Before: `deck.retheme(version_id="dv_01M3…", brand="Co-op")`, where the
agent first called `deck.get_deck` to copy the id.

After: `deck.retheme(brand="Co-op")`
```
Rethemed the published deck (v7; none was named) as Co-op.
```

**A list edited by item**

Before: `edit_design(entities=[…all six entities, each with its
workstreams and labels…])`, to add one label to one entity.

After: `edit_design(entity="Beta", add_labels=["Beta Plus"])`
```
Added label "Beta Plus" to Beta. Other entities unchanged.
```

**A conflict the agent can act on**

Before: `{"rule": "stale_revision", "next_move": "refresh"}`

After:
```
Not applied: slide 4's caption changed since you read it. It is now "Trust held in the South". Redo your edit on that text.
```

**A union flattened**

Before: `anchor: TextAnchor | SlideAnchor` (two nested shapes, ~495 schema
tokens).

After: `slide_number: int | None` and `quote: str | None`, each described
in one line, or no anchor at all on the agent's menu.

**A conditional requirement split**

Before: one `route` tool where `next_hop_id` is required only when
`next_hop_type="router"`.

After: `add_router_route(cidr, router)` and
`add_gateway_route(cidr, gateway)`, each with only the arguments it takes.
