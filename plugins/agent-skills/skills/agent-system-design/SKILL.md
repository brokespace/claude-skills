---
name: agent-system-design
description: Use when designing, changing, or reviewing an LLM agent system in this repo — a charter or system prompt, a tool menu, subagent briefs and architecture, what context each turn carries, finish conditions and budgets — or when judging whether an agent system is good. The rubric agent-diagnosis measures against.
---

# Designing agent systems

An agent does what its system makes likely. It reads the governing text
(system prompt, tool definitions, goal, injected context) on every turn,
acts through its tools, and reads their results. Design every part for a
capable but literal new hire on their first day: they have never seen
your code, can't ask anyone, do exactly what the text says, and guess
wherever it is silent.

The system is good when the agent's shortest correct path — the **ideal
path** — is short, obvious from the text, and cheap to walk, and when
every run leaves a record complete enough to diagnose.

Tools have their own skill, `agent-tool-design`. A decision-model call
(gpt-6-luna screens, filing, ranking) has `decision-model-design`. To find
what went wrong in a run, use `agent-diagnosis`.

## How to use this skill

- **New system or a big change:** work every chapter. Each ends on a
  checklist, and the chapter is done when every box is checked.
- **One part changed** (a charter edit, a new tool on a menu, a new
  subagent): work that chapter, then re-check § Efficiency.
- **Review:** score the system against every checklist. Each unchecked box
  is a finding. **High**: output that can be wrong without anyone noticing,
  a run that can't end, a cost multiplier on every run, a run that can't be
  diagnosed. **Med**: wasted turns or tokens per run. **Low**: polish.

## 1. Architecture

Usually already decided: the system is agentic. Check that the shape fits
the work.

- [ ] Steps that never vary run as code or a fixed LLM call, not as agent
      decisions.
- [ ] A decision that reads wording is a decision-model call
      (`decision-model-design`), or a field the LLM that wrote the wording
      records beside it. That covers an agent's output, a design, a
      research question, a label or questionnaire text. Code decides on
      structure only (numbers, codes, masks, routing). Word overlap,
      regex, stems, slugs and length rules don't decide meaning: LLM prose
      varies too much for them to hold.
- [ ] Subagents exist only where the work splits into independent parts
      that each need a clean context, or where running in parallel pays.
- [ ] Every subagent brief carries an objective, an output format,
      guidance on which tools and sources to use, and boundaries (what is
      not its job).
- [ ] The parent receives a distilled result in the shape its next step
      uses, never a subagent's whole transcript.
- [ ] Effort scales with the task: the prompt says how many subagents and
      calls a small, medium and large request deserves.

## 2. System prompt

- [ ] **Right altitude.** It gives heuristics the agent can apply to cases
      nobody listed, with concrete signals for what good output looks like.
      Neither a brittle tree of if-then rules nor vague advice ("be
      thorough").
- [ ] **Sections**: who the agent is and the background it needs; its job;
      when and why to use each tool; what the output looks like; what
      "done" means.
- [ ] **Minimal.** Every line changes what the agent does. No history, no
      restated defaults, no rules for one past scenario.
- [ ] **One owner per fact.** The prompt says when and why to use a tool;
      the tool description says how. Nothing in the prompt contradicts a
      tool description, and nothing appears in both.
- [ ] **Positive.** It states the behavior wanted. A prohibition appears
      only as a hard guardrail, paired with what to do instead.
- [ ] **Examples** are few, varied and canonical, because the agent copies
      an example literally.
- [ ] **The goal stays visible.** The goal message names the outcome and
      every constraint the agent must honor.
- [ ] **Read as rendered.** You read the prompt exactly as the model gets
      it, with the bag snapshot and every Expertise overlay, and measured
      its tokens.

## 3. Tool menu

Each tool is `agent-tool-design`'s job. The menu as a whole is this one.

- [ ] For each step on the ideal path, an engineer can name the one tool
      that does it.
- [ ] No two tools overlap; tools always called together are one tool.
- [ ] The menu holds only what this agent's job needs (aim for under ~20
      tools).
- [ ] The menu matches the prompt: every tool the prompt mentions exists
      under that name, and every tool has a reason in the job.

## 4. Context

Recall drops as context grows, so every token the agent carries competes
with the ones it needs.

- [ ] You can list what each turn carries (stable prefix, bag snapshot,
      history, results), and each item earns its tokens.
- [ ] Results are sized at the source to what the next step needs; a large
      result is summarized or paged, because it rides in every later
      request.
- [ ] Windowing and compaction never drop a fact a later step needs. Facts
      that must survive live in the bag or the agent's notes.
- [ ] Exploration that would flood the context runs in a subagent that
      returns a distilled result.
- [ ] The stable part of the prompt comes first and stays byte-identical
      across turns, so it caches; the volatile part comes last.

## 5. Termination and verification

- [ ] The agent can tell when it is done: the finish tool's description
      states the criteria, and a refused finish says, in one reply,
      everything still missing and how to supply it.
- [ ] Budgets (turns, tokens, spend) are sized to the ideal path with
      headroom. A budget exit leaves a usable partial result and records
      its reason.
- [ ] Something checks the output against the source (a gate, a checker,
      QC), deterministically wherever the check can be code.
- [ ] A repeated failure escalates or stops the run instead of looping.

## 6. Efficiency

Efficiency is a primary goal: every wasted turn is paid on every run.

- [ ] **The ideal path is written down** for the system's common requests,
      and runs are measured against it (turns, tokens, wall time, cost).
- [ ] **The backend does the work it can.** Apply the backend test to
      everything the agent produces: ids, citations, formatting, derived
      numbers, structure. Each thing the backend could work out, it does,
      and the agent supplies only intent.
- [ ] **No step forces heavy reasoning** for output code could produce. A
      step where agents spend long reasoning or repeated retries is
      redesigned until it is easy.
- [ ] **Independent calls run in parallel**: several tool calls in one
      turn, subagents side by side.
- [ ] **No re-reads.** Results carry what the next steps need, so the agent
      never calls again to recover something it already had.
- [ ] **Reasoning effort fits the slot**, and every GLM slot names an
      effort.

## 7. Observability

A system you can't diagnose is a defective system.

- [ ] Every run records the governing text as sent, each turn's reasoning
      text, each tool call's arguments and its result as the agent read
      it, every error and refusal, and per-call metrics (tokens including
      reasoning, cost, latency).
- [ ] Every subagent run links to its parent run and the spawn call that
      made it.
- [ ] One command or query reads a whole task tree in order. Issue #1187
      tracks this repo's gaps here.

## House rules in this repo

- **Conductor owns the loop.** Provider shaping, the agent loop and almost
  all LLM and agent logic live in conductor; the app adds no wrappers or
  middle-men around it.
- **Least input.** A tool asks for the least input that works and fills in
  what the backend can work out (`AGENTS.md`).
- **Scope comes from the bag.** A Kernel tool argument never carries a
  customer or workspace id (`src/tooling/lint/tool_args_vocab.py`).
- **Lean output.** Every tool result and error the agent reads is
  filtered to text it can act on, never bulk JSON.
- **Charters and prompts are Expertise entries** (`charter/<name>`,
  `prompt/<key>`, `tool_description/<menu>/<tool>`), seeded from
  `bq/expertise/seed/`. A workspace can override any of them, so judge
  the rendered text a run actually sent, not the seed file.

`references/sources.md` lists where these rules come from.
