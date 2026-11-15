# The iteration loop: steps 7–10

`SKILL.md` holds the binding rules; this file holds the trade-offs behind them and the
lessons from projects that used the skill.

## Cutting iterations

- **State the hypothesis before running**, e.g. "with only one year of history, dropping the
  day-of-year feature should stop it from replaying that year's seasonal shape in late
  winter". After the run, check that slice explicitly.
- **Removals are iterations too.** When a feature contributes little, a challenger without it
  tests whether it can go; ties favour the smaller model.
- **Start from the benchmark you must beat.** Giving the model the strong simple baseline as a
  feature makes it learn corrections instead of levels. Replicate the baseline exactly,
  fallbacks included, and verify equality on the evaluation rows.
- **Cheap setup while exploring.** Prefer the cheaper setup (e.g. monthly refits) while
  exploring features; the expensive realistic setup (e.g. weekly refits) is its own iteration
  when a diagnosis points at it.

## One candidate or several per round

"One idea at a time" constrains each candidate, not the number of candidates per round.

| Mode | Information gain | Resources | Follow-up cost and risk |
|---|---|---|---|
| **Single challenger** | one answer per cycle; the next idea builds on its diagnosis | lowest | slowest when many ideas are open |
| **Parallel challengers** (independent single changes, each against the same champion) | several answers per cycle | k × compute or k × wall-clock time | winners must still be combined; the best of k is optimistically selected |
| **Search** (selection, tuning), counts as one challenger | broad, about one dimension | often 10–100 × one candidate | many comparisons inside; confirm outside |

- **Parallel pays off when most candidates are expected to fail:** one round eliminates them
  all, and each winner beyond the first costs one more round later, because gains need not add
  up (two proxies for the same missing driver fix the same errors).
- **More candidates, more false wins.** Do not loosen the confirmation because "one of them
  had to win"; report all k results.
- **Tuning comes late:** a long search on a model with sensible defaults usually yields less
  than a new source of information. Tune once the feature set is stable, or when a diagnosis
  points at capacity or regularisation.
- **The valuation decides:** idle or already-paid compute with the user's time as the
  bottleneck favours parallel rounds, screening and background runs; pay-per-use with a capped
  budget favours single challengers and late, bounded tuning.

## Judging

- A large gain in one partition with nothing in the other usually means the change is
  regime-dependent (see "A feature standing in for missing ones" in
  `references/diagnosis-and-pitfalls.md`).
- Process changes with identical features have no feature-count tie-break; weigh their cost
  explicitly.
- Keep the champion's standing against the baseline visible: it is the number that matters
  for the business case.

## Results layout

- One place names the champion and defines every iteration (name → features, training
  settings); all tables come from it.
- The exploratory shot gets one section, written once.
- One section per open candidate: the change and its hypothesis, results by segment and month
  next to the champion, the protocol tables, the target slice, a summary written after reading
  the outputs.
- Feature importance from more than one measure: split counts favour features with many
  distinct values; add the model's own importance and mean absolute attributions in target
  units.

## Lessons from use

- **A round was run without its plan.** A candidate that followed directly from the last
  diagnosis was built and readied without a round plan, and no valuation of time against
  compute had been recorded; parallel variants were later proposed without a cost assessment.
  The plan is cheap; skipping it hides the choice of mode from the user.
- **The same failure, three times.** Three challengers in a row failed the same guardrail on
  the same slice by the same mechanism. Each needed the full protocol and a new hand-made
  diagnosis, which took most of the cycle. A screening run on that slice and standard
  diagnostics that report the mechanism with every run would have turned one hypothesis per
  working session into several.
- **A review before the run paid off.** An independent review found that a model error
  entered after a nonlinear formula instead of where the domain's laws put it, and estimated
  the outcome on a few entities in minutes, before the full run.
- **Pooled residuals carried low-load behaviour to the extremes.** The use case warned of
  states the training data hardly contain. Residuals pooled over all load levels added what
  they learned at low load to the extremes; flexible and pattern-based models revert beyond
  the training range. A slice beyond the training extremes with proper scores
  (threshold-weighted CRPS, the Brier score of the record event) and plausibility checks as
  gates made this visible.
- **The median summed low.** The median minimises absolute error, but its sum falls short of
  the actual total; the decision needed unbiased totals, so the mean became the point
  forecast and bias a guardrail.
