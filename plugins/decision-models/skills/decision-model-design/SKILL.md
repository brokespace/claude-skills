---
name: decision-model-design
description: Use when designing, changing, calibrating or reviewing a decision-model call in this repo (gpt-6-luna through conductor's decide mode): a screen, filter, filing, rank or dedupe, its questions, the state each question sees, thresholds, how answers combine, and how it is measured.
---

# Designing decision-model calls

A decision model answers named questions about one **state**. A `noul`
question returns one probability; a `choice` question returns a
distribution over named options. It has no tools, no reasoning trace and
no follow-up: it judges exactly the question asked, of exactly the state
given. Design every call for a judge who sees only that state and that
question, and is asked nothing else.

The unit of design is **one criterion, one question, one state**. Code
combines the answers. A good design reads like a checklist a careful
analyst would tick one box at a time. It never reads like one "is this
good?" verdict.

The agent-level rules (altitude, minimal text, efficiency, observability)
live in `agent-system-design`. This skill is the decision-model branch.
`references/findings.md` holds the experiments behind these rules.

## How to use this skill

- **New call or a big change:** work every chapter. Each ends on a
  checklist, and a chapter is done when every box is checked.
- **One question added or reworded:** work § Questions, § State and
  § Calibration for that question.
- **Review:** score the design against every checklist. An unchecked box
  is a finding. **High**: an action taken on a bundled verdict, a must-keep
  item it can drop, thresholds never measured on held-out data. **Med**:
  a state carrying fields its question does not judge, a criterion code
  could decide. **Low**: wording polish.

## 1. Fit: what the model decides and what code decides

- [ ] Everything deterministic is code: counts, a lone figure,
      suppression, a tail box, duplicate figures, a cut's provenance, and
      any catalog fact (two questions sharing a code, a question with no
      wording). The
      model gets only judgements of meaning: relevance to a brief,
      readability, sameness, which of several fits best.
- [ ] A decision that reads wording goes to the model, or to a field
      the LLM that wrote the wording records beside it. That covers LLM
      prose and questionnaire text: a design, a research question, a
      label, an answer. Word overlap, regex, stems, slugs and length rules
      don't decide meaning: LLM prose varies too much for them to hold.
- [ ] Each criterion is written down with its **action**: drop, unlist,
      flag for a fix, file, or order. One criterion, one action.
- [ ] The action fits the remedy. A defect someone can fix (an unreadable
      label, an unnamed filter) is flagged for that fix. Only a defect no
      fix removes (off brief, by construction) drops or unlists.

## 2. Questions

- [ ] **One criterion per question.** A question never asks the model to
      weigh several criteria. "Would an analyst want this, given X, Y and
      Z" becomes three questions, X, Y and Z, combined in code.
- [ ] The `instructions.question` names the state fields it judges in
      backticks, as the seeded entries do ("Does `finding` report the same
      result as `other`…").
- [ ] The question asks for the defect itself, concretely ("is any label a
      formula or a code?"). An abstract quality ("could these go on a
      slide?") is harder for the model to judge.
- [ ] `criteria.true` and `criteria.false` each describe one concrete
      condition. Examples on the false side are all instances of that
      one criterion, never other criteria's failures.
- [ ] A per-item variable (the RQ text, the other candidate) goes in
      through the `instructions` dict (`QuestionText.noul(research_question=…)`).
      The calibrated wording stays fixed.
- [ ] `choice` is used to pick among a bounded pool (ranking, filing to
      one RQ or "none"). `noul` is used for a yes/no judgement. Sameness
      is asked pairwise against what is already kept, in waves
      (`graph/curate/rank.py: walk`).
- [ ] Wording lives in Expertise (`prompt/decision.<area>.<name>`, JSON
      `{instructions, criteria}`) and loads through a question book. It
      never lives in code strings.

## 3. State

- [ ] **The state holds only what the question needs to be judged
      fully, and nothing else.** No background, no field added in case
      it helps, nothing passed along because the caller had it to hand.
      A labels question sees only the labels. A relevance question sees
      the objectives, the RQ texts, the measure and the groups. It does
      not see grids, keys, ids or snake_case internals.
- [ ] Minimal is not missing. The state holds every fact the criterion
      turns on. A named segment carries its definition; a code carries
      its wording.
- [ ] Questions share one spec only when they share the same state.
      Otherwise each gets its own spec; calls are cheap and cached.
- [ ] Context appears only when the criterion needs it. A relevance
      question gets the study's own objectives and RQs, never material the
      product will not have at run time (client decks, labels, a
      reviewer's notes).
- [ ] The state is rendered as the words a reader would see (names, not
      codes). Rendering it is part of the design and is tested.
- [ ] The spec stays well under the size limit. On `DecisionStateTooLarge`
      the caller shrinks the state or halves the pool once, then fails
      the stage.

## 4. Combining answers

- [ ] Combination is code: "any criterion below its own threshold", or a
      small fitted combination, written down with its inputs.
- [ ] Each criterion has **its own threshold**, chosen for its own action.
- [ ] A failed call (outage, 504 after retries) takes the safe default
      for the action: keep the item listed, leave the filing as it was.
      The run records that the call failed.
- [ ] The reason shown to the next reader names the criterion and its
      score, e.g. "judged off-brief: cut (0.08)".

## 5. Calibration and measurement

- [ ] **Ground truth** is a blind, stratified labelled sample, weighted
      back to its strata. Each label carries a reason drawn from the same
      criterion list as the questions. The labelling rules follow the
      product's own policy (what the brief declares, what is asked). A
      label that contradicts the policy teaches the threshold the wrong
      line.
- [ ] **Held out.** Wording and thresholds are tuned on one sample and
      reported on another. A number tuned on the reported sample is not a
      result.
- [ ] Each question is scored against the labels for **its own reason**
      (AUC), and against junk overall. A question that answers the same
      for everything (p≈0 for all) is broken; fix its state or wording
      before using it.
- [ ] **Operating point:** each threshold is set for a bounded loss of good
      items (for example ≤1% useful lost). The report states what that
      threshold catches, and the share of bad items left after.
- [ ] **Must-keep guard:** the items real output used (readouts behind
      accepted slides, nodes a shipped deck cited) are scored, and the
      design loses **none** of them. Each one it would lose is named.
- [ ] Small samples are flagged: at 65 good items, 2% is one item.
- [ ] Thresholds are tied to the model they were calibrated on
      (`calibrated_model`, `resolved_model_matches`, a config version
      hash). Changing the model or any threshold is a regression run.

## 6. Running it

- [ ] Calls go through conductor's decide mode only (`DecideSpec`,
      `Conductor.decide_batch`, the app's `decide_all`/`decide_one`) on a
      named slot. There are no hand-rolled HTTP calls.
- [ ] `cache=True`: answers are deterministic on a content hash, so a
      rebuild re-asks only what changed.
- [ ] Cache misses reserve spend against the customer's cap before they
      run (`reserving`). Concurrency comes from config (≈48–64).
- [ ] Every call is logged with its label, `generation_id`, model, cost
      and whether it was cached.
- [ ] Cost is estimated before a full run. Input-priced at about $0.10 per
      million tokens, about 1,100 card-sized states cost about $0.20 and
      about 20 s.

## House rules in this repo

- **Conductor owns the call.** The app adds no wrapper beyond the
  question book and `decide_all`.
- **Decision models classify, file and rank.** They don't review code or
  documents; reviews go to Opus agents.
- **Local experiments** route by env, never by code, and never print a key
  (see `references/findings.md`, "Running locally").
