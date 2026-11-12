# Sources

Each source is followed by the rules it backs and the file they live in
(SKILL = `SKILL.md`, the others are in `references/`). The size budgets
and the ~3-failure escalation are house defaults, set from these sources
and from our own run, where a 148 KB `graph.question` result pushed the
next round to 242k input tokens. The "in this repo" examples come from the
2026-10 tool audit (removed from `docs/` in 2026-11; see git history).

## Inputs

- Yu et al., *Getting the Parameters Right* (ParamBench, 2026) —
  https://arxiv.org/abs/2608.03071 — only 2.1% of failed calls from 7
  frontier models broke the schema, and the rest were valid but had wrong
  values. The three hardest cases are deep nesting, fields required only
  when a sibling has some value, and values carried over from an earlier
  call's output. Claude Opus 4.7 got 33.4% of NESTFUL calls right with no
  examples. (inputs §intro, §1, §3, §5)
- *ShortcutsBench* (ICLR 2025) —
  https://arxiv.org/abs/2407.00132 — all evaluated LLMs "performed
  exceptionally poorly" at recognizing when they needed input from the user
  or the system; filling parameters from the user's query was harder than
  reusing earlier outputs. (inputs §6)
- Iyer, *Closed-World Resolution Against Tool Hallucination* (2026) —
  https://arxiv.org/abs/2609.19425 — hallucination classes H1–H5,
  including invented or dropped arguments (H2) and the "borrowed
  signature" (H5): valid arguments sent to a wrong but compatible tool,
  which no schema check can catch. (inputs §5, definitions § Name)
- OpenAI, *Function calling* —
  https://developers.openai.com/api/docs/guides/function-calling — "Don't
  make the model fill arguments you already know"; "combine functions that
  are always called in sequence"; enums and object structure "to make
  invalid states unrepresentable" (`toggle_light(on, off)`); the intern
  test; strict mode requires every field and makes optional ones nullable;
  fewer than ~20 functions. (inputs §1, §2, §5; definitions § Tool set;
  SKILL §1)
- Anthropic, *Raising the bar on SWE-bench Verified* —
  https://www.anthropic.com/engineering/swe-bench-sonnet — string
  replacement had "the highest reliability" of the edit strategies they
  tried; "error-proof" tools (always require an absolute path); "much more
  attention should go into designing tool interfaces for models".
  (inputs §2, §3)
- Claude docs, *Text editor tool* —
  https://platform.claude.com/docs/en/agents-and-tools/tool-use/text-editor-tool —
  `str_replace` needs exactly one match of `old_str`, and zero or several
  matches return an error to retry. (inputs §2, §3)
- aider, *Unified diffs make GPT-4 Turbo 3X less lazy* —
  https://aider.chat/docs/unified-diffs.html — lazy elision when asked to
  rewrite whole files; "GPT is terrible at working with source code line
  numbers"; the edit-format principles familiar, simple, high-level, and
  "maximally flexible when interpreting" the model's edits. (inputs §2, §4, §7)
- aider, *LLMs are bad at returning code in JSON* —
  https://aider.chat/2024/08/14/code-in-json.html — every model did worse
  with code wrapped in JSON, with more syntax errors, and strict JSON mode
  didn't help. (inputs §8)
- Claude Code issue tracker, *Edit fails with "File has been modified since
  read"* — https://github.com/anthropics/claude-code/issues/3513 and
  https://github.com/anthropics/claude-code/issues/33856 — shows
  read-before-edit staleness checks, and a check that stays stuck after a
  re-read. (inputs §3)
- Arcade, *Patterns for agentic tools* — https://www.arcade.dev/patterns/ —
  Smart Defaults, Natural Identifier, Parameter Coercion, Constrained
  Input, Mutual Exclusivity, Context Injection, Batch Operation, Idempotent
  Operation; on the output side Response Shaper, Progressive Detail,
  Partial Success, Recovery Guide, and Error Classification. (inputs;
  results; errors)
- Phil Schmid, *MCP best practices* — https://www.philschmid.de/mcp-best-practices —
  top-level primitives over nested dicts (the agent "hallucinates keys,
  misses required fields"); `Literal` types with defaults;
  `{service}_{action}_{resource}` names; 5–15 tools per server; errors as
  instructions. (inputs §5; definitions; errors)
- Idempotent agent tools (industry write-ups, e.g.
  https://tianpan.co/blog/2026/04/19/idempotency-agentic-tool-calling-saga-deduplication) —
  dedupe keys come from the run, step, and action, not from values made up
  at call time; a duplicate returns the original receipt. (inputs §9)

## Definitions

- Anthropic, *Writing effective tools for agents* —
  https://www.anthropic.com/engineering/writing-tools-for-agents — describe
  the tool "to a new hire"; make implicit context explicit; `user_id` not
  `user`; consolidate chained operations (`schedule_event`); namespacing;
  "more tools don't always lead to better outcomes"; evaluate by reading
  transcripts; invalid-parameter errors call for clearer descriptions or
  examples, redundant calls for resizing pagination; "what agents omit …
  can often be more important than what they include"; held-out test sets
  against overfitting. (definitions; inputs §4; SKILL §1, §6)
- Claude docs, *Define tools* —
  https://platform.claude.com/docs/en/agents-and-tools/tool-use/define-tools —
  "extremely detailed descriptions … by far the most important factor";
  what it does, when to use it and when not, what each parameter means,
  and caveats, in at least 3–4 sentences; `input_examples`; consolidation
  with an `action` parameter; service-prefixed names. (definitions)
- Anthropic, *Advanced tool use* —
  https://www.anthropic.com/engineering/advanced-tool-use — a schema
  "can't express usage patterns"; examples raised complex-parameter
  accuracy from 72% to 90%; 1–5 realistic examples, only where the schema
  is ambiguous; five servers take ~55K tokens of definitions; tool search
  raised Opus 4 from 49% to 74%. (definitions)
- Anthropic, *Effective context engineering for AI agents* —
  https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents —
  tools "self-contained, robust to error, and extremely clear"; "if a human
  engineer can't definitively say which tool should be used … an AI agent
  can't be expected to do better"; a minimal viable tool set; lightweight
  handles loaded just in time; context as a finite budget. (SKILL §1;
  definitions § Tool set; results)
- *MCP tool descriptions are smelly!* (2026) —
  https://arxiv.org/abs/2602.14878 — of 856 tools, 97.1% had at least one
  description smell, and 56% had an unclear purpose. Augmenting every
  component gave +5.85 pts median success but +67.46% steps and regressed
  in 16.67% of cases; examples were the least essential component.
  (definitions)
- Block, *Playbook for designing MCP servers* —
  https://engineering.block.xyz/blog/blocks-playbook-for-designing-mcp-servers —
  names, descriptions, and parameters "are treated as prompts"; design
  top-down from workflows; one risk level per tool; bundle
  (`get_issue_info(issue_id, info_category)`); check size before
  returning. (SKILL §1; definitions; results)
- MCP spec 2025-11-25, *Tools* —
  https://modelcontextprotocol.io/specification/2025-11-25/server/tools —
  tool naming rules; input-validation and business errors are results with
  `isError: true` carrying "actionable feedback that language models can
  use to self-correct"; validate inputs, sanitize outputs. (definitions;
  errors)

## Results

- Anthropic, *Writing effective tools for agents* (above) — return
  high-signal fields and names over UUIDs; `response_format`
  concise/detailed (206 → 72 tokens); pagination, filtering, and truncation
  with steering; Claude Code's 25k-token cap. (results)
- Anthropic, *Building effective agents* (Appendix 2, the ACI) —
  https://www.anthropic.com/engineering/building-effective-agents —
  formats close to natural text, no formatting overhead, poka-yoke
  arguments. (results § Format; inputs)
- Anthropic, *Code execution with MCP* —
  https://www.anthropic.com/engineering/code-execution-with-mcp — filter in
  code before anything reaches the model (150k → 2k tokens). (results
  § Size)
- Google ADK, *Function tools* — https://adk.dev/tools-custom/function-tools/ —
  descriptive returns, an explicit status, human-readable error messages
  instead of codes; fewer, simpler parameters. (results; errors; inputs §5)
- Manus, *Context engineering lessons* —
  https://manus.im/blog/Context-Engineering-for-AI-Agents-Lessons-from-Building-Manus —
  deterministic serialization for cache stability; restorable compression
  via handles; "leave the wrong turns in the context". (results § Format;
  errors § Loops)
- Yang et al., *SWE-agent* (NeurIPS 2024) — https://arxiv.org/abs/2405.15793 —
  concise feedback and guardrails; summarized search 18.0% vs iterative
  12.0%; a 100-line window beat the full file; a linting guardrail added
  3.0 pts; collapsing old observations added 3.0 pts. (results; errors)
- Chroma, *Context rot* — https://www.trychroma.com/research/context-rot —
  all 18 models degrade as input grows, and topically close distractors
  hurt most. (results)
- ImprovingAgents, *Best input data format for LLMs* —
  https://www.improvingagents.com/blog/best-input-data-format-for-llms —
  markdown key-value was the most accurate (60.7%) and CSV the least
  (44.3%); a markdown table cost the fewest tokens for decent accuracy.
  Tested on one small model only. (results § Format)
- ToolFailBench — https://arxiv.org/abs/2607.04686 — result-ignore and
  output-fabrication failures; explicit errors and scope/as-of markers
  reduce fabrication. (results; errors)
- *Tool-result truncation: the silent bug* —
  https://dev.to/gabrielanhaia/tool-result-truncation-the-silent-bug-that-makes-agents-lie-3epe —
  silently clipped results get treated as complete; return a "too large"
  error with a narrowing hint instead. (results § Size)

## Errors

- Claude docs, *Handle tool calls* —
  https://platform.claude.com/docs/en/agents-and-tools/tool-use/handle-tool-calls —
  `is_error: true`; instructive errors ("include what went wrong and what
  Claude should try next"); Claude retries 2–3 times on invalid calls;
  tool results are untrusted content. (errors; results § Format)
- *Butterfly Effects in Toolchains* — https://arxiv.org/abs/2507.15296 —
  standardize return formats and improve error feedback to cut cascading
  parameter failures. (errors; results)
- Huang et al., *LLMs cannot self-correct reasoning yet* (ICLR 2024) —
  https://arxiv.org/abs/2310.01798 — models correct well with external
  feedback and poorly without it, and a tool's error is that feedback.
  (errors)
- HumanLayer, *12-factor agents, factor 9* —
  https://github.com/humanlayer/12-factor-agents/blob/main/content/factor-09-compact-errors.md —
  compact errors into context; cap consecutive errors (~3), then escalate.
  (errors § Loops)
