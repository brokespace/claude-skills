# Errors and refusals: recovery instructions

An error is instructions for recovery. The model reads the error text and
nothing else, so the message has to stand on its own. Models correct
themselves well when they get feedback from outside and poorly when they
don't. A tool's error is that outside feedback.

## Where errors go

- **Anything the agent could fix or route around goes back as a tool
  result** marked as an error or refusal (`is_error` / `isError: true`).
  That covers bad arguments, not found, ambiguous, a conflict, a failed
  precondition, too broad, and a busy upstream. Never raise these past the
  loop. A crash teaches the agent nothing, and an opaque failure makes it
  retry blindly or invent an answer.
- **Never report a failure as success.** No `ok` with an empty body when
  the tool couldn't do the work, no fabricated default, and no `ok` for a
  write that changed nothing.
- **A refusal is not an error.** A refusal means the tool worked and said
  no (policy, precondition, limit). An error means something broke. Keep
  the outcomes distinct, so the loop and the logs can tell them apart.

## What an error says

Every error answers three questions, in this order, in the model's terms:

1. **What failed.** Name the argument or action.
2. **Why.** The rule it broke or the state it hit, with the value received
   when that helps.
3. **What to do now.** The exact fix: the valid values, a did-you-mean,
   the tool to call first, or "do not retry; do X instead".

```
wave "2024-Q5" is not in this survey. Waves: 2024-Q1, 2024-Q2, 2024-Q3. Call again with one of them.
```
```
Found 2 questions named "Satisfaction": Q7 (overall) and Q22 (with support). Call again with code="Q7" or code="Q22".
```
```
This asks for 40 segments; the limit is 12. Group them (e.g. by region) or split across calls.
```

Rules:
- **Report every problem at once.** If three fields are wrong, name all
  three, and say whether anything ran. One-at-a-time errors cost one round
  each.
- **Say in words whether to retry**: "Retry once." "Wait and retry." "Do
  not retry this call." Also say what happens if it fails again.
- **No stack traces, exception class names, internal paths, SQL, or bare
  codes.** A stable machine code may go alongside for the logs, but the
  sentence has to work without it. Cap any unavoidable raw error text,
  keeping only its last frames. A user's own Python traceback in
  `run_cell` is the legitimate case.
- **Never leak another tenant's data or internal identifiers** in an
  error.
- **Name real tools and real argument names.** A hint pointing at a tool
  the agent doesn't have is worse than no hint.
- **Repair before refusing.** If the intent is clear, do the work and say
  what you did (`inputs.md` § Accept and repair). Refuse only when the fix
  would be a guess, and then list the candidates.

## Classes of error and the right reply

| Class | Example | The reply tells the agent |
|---|---|---|
| Bad input | wrong type, unknown enum, out of range, unknown argument | each field: got, allowed; the tool's arguments; fix and call again |
| Not found | unknown code or handle | what exists nearby; the tool that lists them |
| Ambiguous | two matches | the candidates and how to pick one |
| Conflict | the object changed since the agent read it | what changed and its current value; whether anything was applied; the call that redoes it |
| Needs the user | only the user knows which brand or wave they mean | the question to ask the user, with the options |
| Nothing to do | a patch that changes nothing | that nothing changed, and which fields it could have changed |
| Too broad | over the size ceiling | how to narrow (a filter, fewer items, paging) |
| Precondition | must build X before Y | the call to make first |
| Transient | busy, timeout, upstream down | wait and retry (once, or after N seconds), or carry on without it |
| Budget / limit | out of time, calls, or spend | stop calling this; finish with what you have |
| Internal bug | invariant broken | do not retry; carry on without it or finish partial; the incident is logged |

## Loops

- An identical repeat of a call that already failed gets a sharper reply
  than the first time. It says the call failed before, and that the agent
  must change its approach.
- After about three consecutive failures of the same tool, the reply tells
  the agent to stop using it for this, and to finish, go another way, or
  ask the user. The harness may escalate on its own.
- Leave failed calls and their errors in the history. They're how the
  agent learns not to repeat them.

## Worked rewrites

These are illustrative shapes, not copies of real tool output.

**A crash that should be an error result.** Before: `KeyError:
'wave_2024_q5'`, raised. The loop logs a traceback, and the agent sees
"tool failed". After (`outcome=error`):

```
wave "2024-Q5" is not in this survey. Waves: 2024-Q1, 2024-Q2, 2024-Q3. Call again with one of them.
```

**Validation one field at a time.** Before: three rounds: `invalid
segment`, then `invalid metric`, then `limit too high`. After, one round:

```
3 problems — nothing was run:
- segment "Gen Z" is not a variable here. Did you mean "Age group" (has an 18–24 band)?
- metric "top3box" is not supported. Use top2box, mean, or pct.
- limit 50 is over the cap of 12.
```

**A refusal whose next move is a rule name.** Before:
`{"refusal": {"rule": "precondition_failed", "next_move": "storyboard_required"}}`.
After:

```
No storyboard yet, so there is no slide 3 to edit. Call storyboard.create first, then retry this edit.
```

**A not-found that names no way forward.** Before: `check the beat id`.
After:

```
No slide has id "bea_01M3…" on this storyboard. locate_beat finds a slide by its number or headline; get_storyboard lists them all.
```

**A repeat of a failing call.** The second identical call after an error
gets:

```
Same call, same error as your last try: wave "2024-Q5" does not exist. Retrying will not change it. Pick a listed wave or continue without the wave filter.
```
