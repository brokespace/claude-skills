# Attributing a failure to a layer

Walk the layers in this order. For each, the symptoms say where to look,
the confirming quote says what proves it, and the fix says where it goes.
A symptom can point at several layers; the quote decides.

## 1. Task or data

The goal is ambiguous or impossible, or the data is missing or wrong.

- **Symptoms:** the agent's reasoning debates what the user meant; correct
  reasoning over wrong numbers; every run fails the same way whatever path
  it takes. Zero passes across several runs usually means a broken task,
  not an incapable agent.
- **Confirm:** read the input and the source data directly.
- **Fix:** the task, the data, or a question back to the user.

## 2. System prompt

The governing text is missing, wrong, contradictory, buried, pitched too
vague or too rigid, or carries a misleading example.

- **Symptoms:** the agent cites an instruction and applies it the way it
  reads, not the way it was meant; it invents a procedure where no text
  covers the step; it copies an example literally; it follows one of two
  conflicting rules.
- **Confirm:** quote the text, or show that no text covers the step. Would
  a literal new hire given only this text do what the agent did? If yes,
  this layer owns it.
- **Fix:** reword or delete first. See `agent-system-design` § System
  prompt.

## 3. Tool description vs. prompt

The prompt and a tool description disagree, or the description is wrong
about what the tool does.

- **Symptoms:** reasoning weighs two instructions against each other;
  arguments match the description but not the prompt's intent; invalid
  argument errors on first use of a tool.
- **Confirm:** quote both passages side by side.
- **Fix:** one owner per fact. Usage details belong in the tool
  description; when and why to use the tool belongs in the prompt.

## 4. Tool result or error

The result is wrong, misleading, bloated, silently truncated, or empty
without saying why; or an error points away from the real fix.

- **Symptoms:** the agent's next move follows the result or error
  faithfully and goes wrong; the same call is repeated after an error; a
  guess follows an empty result; input tokens jump after a tool round; the
  agent re-calls to get more of something.
- **Confirm:** quote the result as the agent read it (from the request
  history, not the tool's typed result) and the reasoning that acted on
  it.
- **Fix:** `agent-tool-design` §4 (results) or §5 (errors).

## 5. Tool design

The tool shouldn't exist in this shape: it asks for what the backend
knows, makes the agent assemble structure by hand, overlaps a sibling, or
splits one step across two tools always called together.

- **Symptoms:** heavy reasoning spent composing one argument; repeated
  failures at one step despite clear text; the agent alternating between
  siblings; the same pair of calls every time.
- **Confirm:** the backend test — list what the agent produces at this
  step and mark each item the backend could work out itself.
- **Fix:** `agent-tool-design` §1–§2; repeated failures here fire the
  redesign trigger.

## 6. Harness

The loop around the agent lost or withheld something: windowing,
compaction or truncation dropped a fact; a budget, guard or spend cap
ended the run; parallel calls raced; a timeout cut a call.

- **Symptoms:** the agent forgets a fact it had, repeats a completed step,
  or asks for something already returned; the run ends abruptly; the run
  ends with `budget` or `spend_cap`; a stall shows up as a wall-time gap.
- **Confirm:** show the fact present in an earlier request and absent from
  the failing turn's request, or quote the exit reason.
- **Fix:** what the loop carries or how it ends. See `agent-system-design`
  § Context and § Termination.

## 7. Model

Reached only when, at the failing turn, the governing text was clear and
consistent, the tool results were correct, and the needed context was
present, and the agent still went wrong in more than one run.

- **Confirm:** all three quotes, plus a second run failing the same way.
- **Fix:** a proposal to the user for a different model or reasoning
  effort on that slot.

## Multi-agent checks

When a subagent's output is wrong, read its brief before blaming it: did
it get an objective, an output format, tool guidance and boundaries? Then
check the hand-offs in both directions:

- the parent withheld a fact the subagent needed;
- the parent ignored or overwrote what a subagent returned;
- two subagents duplicated each other's work;
- an agent's stated plan contradicts the action it took next.

## Ending checks

- **Ended too early:** a finish accepted before the work was complete, or a
  verification that never ran.
- **Never ended:** the agent couldn't tell it was done, or a finish refusal
  didn't say what was missing.
- **Wrong verdict:** a check passed bad output or refused good output.
