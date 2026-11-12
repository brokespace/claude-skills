# Summarizing reasoning with Haiku

Reasoning is the best evidence in a run, and too long to read raw in full.
Haiku subagents compress it into quotes you can check; you keep the
judgment.

## What goes to Haiku

- **Every reasoning block over ~4k tokens.** In `bq trace --out`'s
  `index.md`, a block over ~4k tokens is one turn too large to share a
  block; read its reasoning size there. One Haiku per block; dispatch them
  in parallel.
- **The whole run, for an overall summary.** A run under ~100k tokens goes
  to one Haiku. A larger run is split at turn boundaries into chunks under
  ~100k, one Haiku per chunk, each told which turns it covers.
- **Never the governing text.** You read the system prompt and tools
  yourself (step 2); give Haiku the short context below instead.

Hand Haiku block file paths from the trace, never pasted text, so the transcript
stays out of your own context.

## The brief

Dispatch with `model: haiku`. Fill the brackets.

```
You are summarizing part of an LLM agent's run for someone diagnosing it.
Read-only: read the files below and report. Do not run anything else.

The agent's job: [goal, one or two sentences].
Its tools: [name — one line each].
A short correct path would be: [the ideal path, numbered].
Files: [paths to turns/NNN.md], covering turns [a–b].

Return, in this order:
1. Summary (5–10 lines): what the agent was trying to do in these turns
   and how it went.
2. Confusion: every point where it hesitates, backtracks, contradicts
   itself, re-derives something it already had, or reasons at length about
   how a tool, argument or instruction works. For each: turn number, a
   verbatim quote of at most 3 lines, and a one-line gloss.
3. Beliefs: what it believed each tool, argument or instruction meant,
   quoted with turn numbers, wherever it says so.
4. Plan changes: each place it changed course, with the turn and the
   reason it gave.
5. Guesses: values or facts it assumed without checking, quoted.
6. Your interpretation: what you think is going wrong and why. Label this
   section as interpretation.

Quotes are verbatim, never paraphrased. Write "none" for an empty section.
```

## Using what comes back

The summary points you at spots; it is not evidence. Open the raw turn
around every quote in sections 2–5 and read it yourself before a quote
enters a finding. Haiku's interpretation is a lead to test like any other
candidate cause. The root cause is your call.
