# Reading and running agents in this repo

## The agent systems

Every loop runs through `TaskContext.loop(charter, goal, tools=<menu>, slot=…)`
(`bq/task/context.py`). The charter is Expertise `charter/<name>`, seeded
from `bq/expertise/seed/charters/*.md`. The system prompt each round is the
charter plus a tool-name index, followed by the rendered bag snapshot
(`bq/task/loop_context.py`). Tool definitions come from the menu
(`KernelToolSource`), with Expertise `tool_description/<menu>/<tool>`
entries replacing docstrings.

| System | Task kinds | Prompt and tools |
|---|---|---|
| Research | `research_request` (from `research.ask`) | `bq/research/agent/` — `prompts.py` assembles role text, survey API text and catalog index from Expertise `prompt/research.*`. Tools: `run_cell` (notebook) alone, whose `ship()` submits the answer (a non-survey request's passages loop has the `finish` tool). Slots `research.researcher`, `research.checker`, … |
| Storyboard | `create_storyboard`, `edit_storyboard` | `bq/task/recipes/storyboard/` — charters `storyboard-spine`, `-writer`, `-editor`, `-closing`, `-slide` and `-core`. Tools `bq/storyboard/tools/`. |
| Deck | `build_deck`, `refresh`, `retheme` | conductor sandbox agent (`bq/task/runtimes/build_spec.py`), tools `bq/deck/tools/`. |
| Datasets | `dataset_turn`, `dataset_onboard`, `dataset_wave` | charters `dataset-onboarding*`, tools `bq/datasets/tools/`. |
| Analyst chat | `turn` | charters `analyst_chat`, `expertise_chat`. |
| Study | `pre_research`, `study_stage` | `bq/task/recipes/study/`, slots `study.*`. |

## Where a run's record lives

Every run is recorded once, through conductor's typed record: in Postgres
inside bq, in a SQLite file (`CONDUCTOR_RUN_STORE`, default `.data/runs.db`)
when conductor runs standalone. `bq trace <task_id or run_id>` reads either
(`--store <file>` for SQLite) and prints, or with `--out DIR` writes, the
whole tree as a transcript; `/api/tasks/{id}/trace` and
`/api/runs/{id}/trace` serve the same `RunTrace` as JSON.

- **What the record holds:** per run its identity (`task_id`, `role`,
  `parent_run_id`, `parent_tool_call_id`, outcome); per model call the
  system prompt, tools and new messages as sent (the request rebuilds
  exactly), the response, the reasoning text, cache hints, tokens, cost,
  `call_key` and the upstream provider; per tool call its arguments and the
  reply exactly as the model read it. A body the store lacks prints
  `UNRESOLVED`, never blank.
- **Reasoning:** a turn with reasoning tokens but no reasoning text is an
  observability finding to report; never guess what the agent thought.
- **Subagents and child Tasks** are placed under the tool call that started
  them (`parent_tool_call_id`, `task.spawned_by_tool_call_id`).
- **Postgres** (`psql` against the local dev database, credentials from the repo `.env`;
  research e2e scratch DBs have their own port), for queries the trace does
  not answer:
  - `run`, `llm_call`, `agent_tool_call`, `content` — the record's tables
    (bodies are inline columns or `content` rows by hash).
  - `task` — `parent_id` and `root_id` give the task tree.
  - `task_event` — the task's log ordered by `seq`: stages, tool lines,
    refusals, questions.
  - `tool_call` — Kernel tool calls with `args`, `result`, `refusal`,
    `error`, `ms`, joined to the run by `agent_tool_call_id`.

## Launching a run

The API runs on port 8000 (pm2 `bq2-api`, hot reload). Start a run through
its route; `scripts/storyboard_compare/bq2_side.py` shows how to derive a
dev JWT and poll a task to completion.

- Research: `POST /api/workspaces/{id}/research/ask`.
- Storyboard: `POST /api/workspaces/{id}/storyboard/create` or `/edit`.
- Deck: `POST /api/workspaces/{id}/decks/build`.
- Cancel: `POST /api/tasks/{id}/cancel`.

The research e2e harness lives in `~/.cache/rr_e2e` (`/tmp/rr_e2e` is a
symlink to it). `run_one.sh`, `run_batch.sh` and `run_q.sh` run seeded EE
research questions with `CAP_USD` and a spend watchdog; they swap
`models.glm.json` into `src/backend/config/models.json` and restore it on
exit. The storyboard harness is `~/.cache/ee_story/story.py`.

## Watching a live run

- Poll `task_event` for the task, ordered by `seq`.
- `uv run bq trace <task_id> --follow` prints each turn once it is
  complete, until every run ends.
- In `~/.cache/rr_e2e`: `watch_steps.py` streams each model call as it
  lands; `watch_calls.py` stops a run on the first error, refusal, slow
  call or spend overrun; `watchdog.sh` kills a run when the key's spend
  passes `LIMIT`; `activity.sh` prints today's gateway spend per model.
- Use the Monitor tool with an until-loop to wait on a condition rather
  than sleeping.

## Spend and models

- A post-hoc diagnosis spends nothing. A live run happens only when the
  user asked for one in this request; their ask covers that run plus one
  verification re-run.
- Before launching, state the system, the slot→model mapping and an
  estimated cost. The mapping resolves from workspace override
  (`llm_slot_override`), then admin default (`llm_slot_default`), then
  `src/backend/config/models.json`.
- Every call goes through the Hermes gateway: `OPENROUTER_BASE_URL` with
  `OPENROUTER_API_KEY`.
- Large slots run on `z-ai/glm-5.3`, mid on `google/gemini-3.8-flash`,
  small on `openai/gpt-6-luna`; no request ever goes to an Opus model.
  Every GLM slot names a reasoning effort, `high` for agent runs.
- Budget against the key's real spend (`GET {base}/key/info`), which runs
  about 10–50× the per-call `usage.cost`.
