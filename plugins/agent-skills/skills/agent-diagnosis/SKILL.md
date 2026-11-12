---
name: agent-diagnosis
description: Use when diagnosing an LLM agent in this repo — watching a live agent run, reading a finished run's trace, explaining why an agent did something, looped, stalled, misused a tool, or ran slow or expensive, or sorting many runs into failure categories. Covers research, storyboard, deck, datasets and every other conductor loop, including their subagents.
---

# Diagnosing agent systems

In this skill **Claude** is you, the reader. **The agent** is any LLM agent
you are analyzing: the research agent, a storyboard section worker, a
subagent one of them spawned.

An agent's behavior is evidence about the system around it. The agent did
what its prompt, its tools' descriptions and results, and its context made
likely. Your job is to find which of those made the failure likely, prove
it with quoted text, and fix it there. Model capability is the **last**
explanation, reached only once the other six layers are ruled out by
quotes.

Two companion skills hold the rubric. `agent-system-design` says what a
good agent system looks like; `agent-tool-design` says what a good tool
looks like. Load the one whose layer a finding lands in before you write
its fix.

**Efficiency is first-class.** A run that reached the right answer in 40
turns where 8 would do has findings, and they rank by their cost, not
behind every correctness nit.

## Modes

- **One run** (default): one run, a task tree with subagents, or a live
  run. Work steps 1–9.
- **Many runs**: an eval or parity sweep, "look at these 20 runs". Read
  `references/many-runs.md`; it reuses steps 2–6 per run.

What happens after the diagnosis is the user's choice:

- **Diagnose and propose** (default, and whenever the request is
  ambiguous): stop at the report, step 9.
- **Closed loop**: only on explicit words such as "fix it", "iterate
  until", "make it pass". Then step 8 applies fixes and re-runs.

## 1. Get the run

`references/reading-runs.md` covers where runs live, how to find a task's
runs, launching, and live watching in this repo.

- **Finished run**: write the task's whole run tree with
  `uv run bq trace <task_id> --out /tmp/agent-trace/<task_id>` (or a run
  id; `--store <file>` for a standalone SQLite run store). `index.md` holds
  the metrics tables and the block list; each `part-NN/block-NNNN.md` holds
  whole turns (system prompt and tools once per run, then per turn the new
  input, reasoning, response, tool calls with arguments, each reply as the
  model read it, errors and refusals), with subagent and child-Task runs
  nested after the turn that started them.
- **Live run**: only when the user asked for one in this request. Before
  launching, state the system, the slot→model mapping and an estimated
  cost; the user's ask covers that run plus one verification re-run. Watch
  it as it streams. When you see a failure, judge it: if the agent will
  recover quickly on its own, let it run and fix the cause afterwards; if
  it will derail or stall the run, stop the run now.

**Done when** you hold the task id, every run id in the tree, and each
run's role (slot or charter).

## 2. Read the governing text, in full

The **governing text** is everything the model was given, exactly as sent:
the system prompt, every tool definition, the goal message, and anything
injected (the bag snapshot, catalog index, retry notes). For a subagent it
also includes the parent's brief, which is the arguments of the spawn
call.

Read it from the dump (`system.md`, `tools.md`, `goal.md`), never from the
source template. The template is expanded, overlaid by Expertise entries
and joined with a live bag snapshot, so the model's copy differs from it.
Read every line yourself; this step is never delegated. A 40 KB prompt is
about 10k tokens.

**Done when** you can say, for each tool, when the governing text tells
the agent to use it, and you have a written list of every place the
prompt and a tool description disagree, or "none".

## 3. Write the ideal path

Before reading what the agent did, write the **ideal path**: the shortest
sequence of calls a competent agent with this governing text and these
tools would make for this goal. Note expected turns, which calls can run
in parallel, and the user's budget for this system if they gave one.

If you cannot write a short ideal path, that is a design finding: the
step that resists it is a redesign candidate (step 7).

**Done when** the ideal path is written down as numbered calls.

## 4. Read the run

**Metrics first.** Read `metrics.md` for each run: turns, input, output
and reasoning tokens per call, tool outcomes, wall time, cost. Mark every
outlier:

- a call whose reasoning tokens are several times the run's median (the
  agent was working something out — often confusion);
- a tool called again with near-identical arguments;
- an error or refusal followed by the same call;
- an input-token jump after a tool round (a bloated result);
- a long gap in wall time.

**Then read every turn, in order, from the first.** Reasoning is the most
important evidence in the run: it shows what the agent believed an
instruction or a tool meant, and where it got lost. Every reasoning block
over ~4k tokens goes to a Haiku subagent per
`references/reasoning-summary.md`; read the raw text around each spot it
flags yourself.

Find the **first failure**: the earliest turn where the agent leaves the
ideal path, or where what it believes stops matching the truth. Later
failures are usually its cascade. Record them, and attribute the first.

Before moving on, check each of these:

- **The output against the source data.** A confident, well-formed final
  answer can be wrong. Check its numbers and claims against what the tools
  returned.
- **Every subagent's brief**, read before judging that subagent's output:
  objective, output format, tool guidance, boundaries.
- **The agent's context at the failing turn.** Was the fact it needed
  there, or had it been windowed, compacted, truncated, or never given?
- **The end of the run.** Could the agent tell it was done? Read the
  finish call, any finish refusals, and budget or spend exits.

**Done when** every turn is read or summarized, every outlier is
explained, and the first failure is named with its turn number.

## 5. Attribute it to a layer

Walk the seven layers in order. `references/attribution.md` lists, per
layer, the symptoms that point there and the quote that confirms it.

1. Task or data
2. System prompt
3. Tool description vs. prompt
4. Tool result or error
5. Tool design
6. Harness (context, windowing, budgets, guards)
7. Model

A cause you keep needs two quotes: the agent's own reasoning or call at
the failing turn, and the governing text that bears on it. "No text covers
this" is a valid second quote, and it is a prompt-layer finding.

When the agent misused a tool, the prompt and the tool's description are
suspects before the tool itself: read what both told the agent about that
tool, and ask whether a literal new hire given only that text would have
done what the agent did. If yes, the text is the cause.

List every candidate cause that fits, then test each against the trace.
A cause is established when it explains all the evidence and quotes rule
out the others; one plausible cause is a hypothesis.

**Done when** each finding carries a layer, its two quotes, and the reason
each earlier layer doesn't explain it.

## 6. Efficiency pass

Compare the run against the ideal path step by step. Each gap is a
finding, costed per run in turns, tokens, seconds and dollars:

- turns or calls beyond the ideal path;
- serial calls that could run in parallel;
- re-reads and retries;
- results larger than the next step needs (they ride in every later
  request);
- subagents spawned for work one call would do;
- reasoning spent producing something the backend could produce.

Rank every finding from steps 5 and 6 in one list. Corrupt or wrong output
comes first; everything else ranks by impact × frequency (cost per run ×
how often the step runs).

## 7. Choose each fix

- **Fix at the layer the cause lives in.**
- **Remove or clarify before adding.** A fix that adds a prompt paragraph
  or a new tool states why removal or rewording can't work.
- **Write prompt fixes as general principles** the agent can apply to
  cases you haven't seen. A rule naming this run's scenario overfits.
- **One owner per fact.** When the prompt and a tool description disagree,
  pick the place that owns the fact and delete it from the other.
- **Architecture changes** (merge agents, replace an agent with a fixed
  workflow) are proposals.
- **Model and reasoning-effort changes** are proposals only, and come
  after prompt and tool fixes.

**Redesign trigger.** Stop patching a step and redesign it when any of
these holds:

- two patches at the same step have failed;
- the same step fails across runs or across agents;
- one trace shows heavy reasoning or retries spent producing something the
  backend could produce.

To redesign, list everything the agent must produce at that step and
apply the **backend test** to each item: could the backend work this out
itself? Propose the step with the agent supplying only intent.

*Worked example.* Research agents kept failing the answer-submission step,
and patch after patch to its refusals and wording didn't stop it. The
agents had to write complex citations by hand. The backend already held
everything those citations pointed at. The fix was to make submission
easy: the agent states the answer, and the backend handles the citations.

## 8. Fix and verify (closed loop only)

- Rewrites of a system prompt, charter or tool description are shown to
  the user before they land. Plain bugs (a crash, a dropped value, an
  error that points away from the fix) are fixed directly.
- Verify with one re-run of the same scenario. Compare the metrics before
  and after, confirm the first failure is gone at its turn, and confirm
  nothing new fails earlier. A run that passes for a reason other than
  your fix is not a verification.
- Two fix attempts at the same failure with no new information means stop:
  apply the redesign trigger or report the blocker.

## 9. Report

Report in chat using `references/report.md`: ranked findings, each with
symptom, first failing turn, the two quotes, layer, root cause, fix,
verification and confidence, then the metrics table. A finding you did not
fix becomes a GitHub issue on the repo, per `AGENTS.md`.
