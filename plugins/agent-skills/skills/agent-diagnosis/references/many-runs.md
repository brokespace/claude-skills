# Many runs: building a failure taxonomy

Use this mode for an eval or parity sweep, or any request to look across a
set of runs. The categories come from the runs; start with none.

1. **Sample.** Take the runs in scope. If there are more than ~30, sample
   across the dimensions that matter (question type, dataset, outcome,
   cost) rather than taking the newest.
2. **Read the governing text once per system** (step 2 of the main skill)
   and write its ideal path per scenario (step 3).
3. **Note each run.** For every run, dump it, read it (step 4, with Haiku
   for long reasoning), and write one free-text note: pass or fail, the
   first failure in plain words as an observation ("the worker re-ran the
   same crosstab three times after an empty result"), and its efficiency
   gap against the ideal path. Note what you saw, not why.
4. **Group.** After 20–30 notes, group them into 5–10 categories. Each
   category is distinct, specific enough to apply consistently, and
   carries two or three example quotes with run ids.
5. **Show the user.** Present the categories with their examples and wait
   for the user to confirm or reshape them before labeling.
6. **Label every run** against every confirmed category, pass or fail.
   Keep reading new runs until about 20 in a row add no new category.
7. **Rate and rank.** Count each category's failure rate. Rank by impact ×
   frequency, with efficiency categories costed in turns, tokens and
   dollars per run.
8. **Attribute the top categories.** Take two or three runs from each and
   work steps 5–7 of the main skill on them, so each category ends with a
   layer, quotes and a fix.

Report the category table (name, definition, rate, cost, layer, fix), then
the per-category findings in the main report format.
