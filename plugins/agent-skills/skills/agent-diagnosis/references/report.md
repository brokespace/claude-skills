# Report format

Findings first, ranked: corrupt or wrong output, then everything else by
impact × frequency. One block per finding:

```
### F1 — <one-line symptom>  [layer: <1–7 name>] [confidence: high|medium|low]

- First failing turn: <run id / turn N> (<slot>)
- The agent: "<verbatim reasoning or call at that turn>"
- Governing text: "<verbatim instruction, description, result or error it acted on>" — or "no text covers this step"
- Root cause: <one or two sentences; why earlier layers don't explain it>
- Cost: <turns / tokens / seconds / $ per run, and how often the step runs>
- Fix: <what changes, at which layer, in which file or Expertise entry>
- Verify: <the re-run and what it must show>
```

Then the metrics table, per run and in total:

| run | slot | turns | tool calls (errors) | in / out / reasoning tok | wall s | $ | ideal-path turns |
|---|---|---|---|---|---|---|---|

Close with:

- **Observability gaps**: anything you couldn't see (missing reasoning
  text, unrecorded final call), each with its effect on confidence.
- **Not fixed**: each with its GitHub issue link.
- **Applied** (closed loop only): each fix and its before/after metrics.
