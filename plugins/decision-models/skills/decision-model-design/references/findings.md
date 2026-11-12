# Findings behind the rules

Each rule in `SKILL.md` traces to one of these experiments. All used
gpt-6-luna through conductor's decide mode.

## Relations screen (2026-10-27, client study)

- The stored curate screen scores predicted deck use at chance (AUC
  ~0.5). The "relevant" tag ran inverted (0.35).
- Per-node usefulness questions plateaued at AUC ~0.6. Deck use is limited
  by slots and is redundant, so per-node usefulness is a weak lever.
- One high-precision question ("an incidental comparison") dropped 15% of
  nodes and lost none of the 174 the deck used. This supports **narrow
  criterion, high-precision threshold**.
- Filing was the big lever: one `choice` question over all RQs plus "none"
  picked a question in the deck section's objective 75% of the time (top-2:
  86%). This supports **choice for picking among a bounded pool**.

## Readout junk screen, bundled (2026-10-29, A6)

- **Setup:** readout cards plus the study's objectives and the filed RQ
  texts. Questions were bundled: "an analyst would want this… not if
  incidental, unreadable, hidden-filtered or empty".
- **Agreement:** AUC 0.76 held out.
- **Operating point:** at 2% useful loss it caught 25–28% of listed junk.
- **Failures:**
  - It dropped the only readout behind 3 accepted client slides: a table
    with formula labels, which a fix would rename, and a tenure-filtered
    trend it read as arbitrary. This supports the **must-keep guard** and
    the **action fits the remedy** rule.
  - The labels question gave p≈0 for every card when it saw the whole
    card, with its keys line and snake_case keys. This supports the
    **minimal state per question** rule.
  - It caught mostly shapes code already handles, such as lone figures.
    This supports the **code decides what is deterministic** rule.
- **User guidance afterwards:** decision models are good at single
  questions, so ask one question per criterion and combine the answers.
  This is the rule at the centre of the skill.

## Readout junk screen, one question per criterion (2026-10-29, A6b)

- **Setup:** each criterion got its own question and only its own fields.
  Thresholds were set on A4 at ≤1% useful loss each, and reported on A5.
- **What worked:** a labels question worded as "is any label a formula or a
  code?" flags 3% of listed readouts at 92–100% precision. The first
  wording ("could every label go on a slide?") gave p≈0 to readable long
  labels. This supports **asking for the defect, concretely**.
- **What failed:** the cut questions ("would an analyst split `measure` by
  `cut`?", "could the group change the answer?", "are the groups defined
  by the measure?").
  - They dropped from AUC ~0.8 on A4 to ~0.65 on A5.
  - They lost 10% of useful readouts in one client study.
  - They would unlist 5 readouts behind accepted slides.
- **Causes:**
  - **The labels disagreed with the policy.** The labelling rules called a
    cut incidental when the brief named it. Ground truth has to follow
    the same rule the product follows (see the ground-truth box in § 5).
  - **The state lacked a definition the question needed.** Segment names
    (client-specific persona names) are opaque without their definitions. A
    minimal state still has to hold every fact the criterion turns on.
  - **Code knew more.** "Groups defined by the measure" and "hidden
    allocation variable" are catalog facts: question codes and whether a
    question has wording. Code should decide them.

## Running locally

Experiments run outside the app through conductor:
- `decide_run` with `ProviderPool.from_env()`;
- an on-disk body store as the cache;
- `ModelRef(provider="openrouter", model="gpt-6-luna")`.

Route to OpenAI's endpoint by env: set `OPENROUTER_DECISIONS_BASE_URL` to
`https://api.openai.com/v1`, and `OPENROUTER_DECISIONS_API_KEY` from
`OPENAI_API_KEY` in the repo-root `.env`. The staging gateway rejects
local keys.

Expect a few HTTP 504s per few thousand calls, even after retries. A
cached rerun fills them.
