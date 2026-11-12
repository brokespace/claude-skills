# Sources

- Anthropic, *How we built our multi-agent research system* — subagent
  briefs (objective, output format, tools, boundaries), effort scaling,
  parallel calls, watching agents step by step with their exact prompts and
  tools. https://www.anthropic.com/engineering/multi-agent-research-system
- Anthropic, *Effective context engineering for AI agents* — prompt
  altitude, minimal prompts, bloated tool sets, context rot, compaction,
  subagents returning distilled results.
  https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents
- Anthropic, *Building effective agents* — simplicity, workflows vs.
  agents, agent-computer interface design.
  https://www.anthropic.com/engineering/building-effective-agents
- Anthropic, *Writing effective tools for agents* — transcript metrics,
  redundant calls and parameter errors as signals, held-out evaluation.
  https://www.anthropic.com/engineering/writing-tools-for-agents
- Anthropic, *Demystifying evals for AI agents* — reading transcripts,
  pass@k vs. pass^k, a 0% pass rate signalling a broken task.
  https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents
- Cemri et al., *Why Do Multi-Agent LLM Systems Fail?* (MAST) — 14 failure
  modes across specification, inter-agent misalignment and verification;
  most failures trace to system design. https://arxiv.org/abs/2503.13657
- *Where LLM Agents Fail and How They Can Learn From Failures*
  — one root-cause error cascades; find the first critical error.
  https://arxiv.org/abs/2509.25370
- *Repair or Resample? Rethinking Failure Debugging in LLM Multi-Agent
  Systems* — most failures are systematic; plain reruns rarely repair
  them. https://arxiv.org/abs/2608.25920
- Hamel Husain, *AI Evals FAQ* and the `error-analysis` skill — note the
  first upstream failure, open then axial coding, categories that emerge
  from traces, saturation. https://hamel.dev/blog/posts/evals-faq/ ,
  https://github.com/hamelsmu/evals-skills
- User rulings, 2026-10-01 (the session that designed these skills):
  efficiency as a primary goal; the seven-layer attribution order; the
  redesign trigger and its answer-submission example.
