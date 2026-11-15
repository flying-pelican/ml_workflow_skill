---
name: ml-model-iteration
description: Disciplined, incremental improvement of a predictive model (forecasting models on panel/time-series data, but also other supervised ML) through champion/challenger iterations with a pre-agreed, data-driven evaluation protocol. Use this skill whenever the user wants to improve, iterate on, tune, or add/drop features of a forecasting or ML model, compare a new model version against the current one or a baseline, decide whether a change is "really better", set up a backtest, cross-validation or train/validation/test split, choose between random, grouped, time-based or spatial validation, translate a business use case into model requirements (e.g. a smooth, monotonic price response for price optimisation), or diagnose why a model loses to a simple benchmark — even if they don't say "champion", "challenger" or "iteration". Also use it when the user asks how to evaluate or select between models, or wants a careful, step-by-step modelling workflow instead of a big one-shot model.
---

# ML model iteration: champion, challenger, protocol

Improve a model one verifiable step at a time. The usual failure of model development is not
a lack of ideas but believing in gains that are noise, leakage or specific to one period, and
stacking further changes on top of them. Every step lets the user see what changed, why, and
whether it is real.

**The user decides** every round plan, every promotion and every commit. Present evidence and
a recommendation; do not run a round, promote, commit or publish on your own.

## How to use this skill

- **Project notes hold the agreements and the state.** Record the agreements of every step
  (use case, metrics, acceptance checks, guardrails, split, the valuation of the user's time
  against compute, conventions for code and history) and the state (champion, open
  candidates, current step) in the project's notes; update the state after every decision.
  At the start of a session, or when unsure what was agreed, read the notes first.
- **Rules here, reasons in `references/`.** Before a step, if the content of the reference
  named for it is not in your context, read it.

| Reference | Read for |
|---|---|
| `references/preparation.md` | steps 0–6: use case, interview, cycle length, exploration, baseline, exploratory shot |
| `references/evaluation.md` | step 3, and close calls in step 8: information cutoff, split design, partitions, block bootstrap, metrics |
| `references/iteration-loop.md` | steps 7 and 10: one or several candidates, results layout, lessons from use |
| `references/diagnosis-and-pitfalls.md` | step 9, and whenever a gap persists or a result looks too good |

Preparation (steps 0–6) is done once per project; the loop (steps 7–10) for every
challenger. Do not start the loop before the preparation is complete.

## Every round: the iteration loop (steps 7–10)

**7. Plan the round. The user agrees before anything runs, also when the candidate seems
obvious.**
- **One change per candidate**, with one hypothesis and the target slice it should improve.
  A change is one feature, one feature group that tests one hypothesis, one removal, or one
  process change (refit cadence, loss, model size) with the features unchanged.
- Derive the candidate from a diagnosed mechanism (step 9) or the exploratory shot's ranked
  list, not from a list of plausible features.
- If the change uses a named method, look up its canonical form in primary sources and name
  every deviation with its reason. Have deviations, and runs that are expensive or follow
  repeated failures, reviewed independently before running (a fresh agent or a colleague;
  the user decides who).
- Choose the mode by expected information gain per run time and cost, under the user's
  recorded valuation: one challenger; several independent challengers against the same
  champion; or one search (selection, tuning), bounded before it starts and kept inside the
  selection partition. Variants of one idea are a search, not parallel challengers.
- **Screening** is a reduced run (one slice, a subsample, a coarser cadence), in both
  partitions, that may reject a candidate, never promote it: it cannot show harm elsewhere,
  so survivors get the full protocol. Screen first when earlier candidates failed on the
  same slice, whether they were rejected or not adopted.
- State the plan: candidates with hypotheses and target slices, the mode and why, the
  expected run time and cost, and what a win or a loss triggers next.

**8. Run and judge.**
- Same rows, same information cutoff and the same scoring code for every method.
- **Acceptance checks first:** absolute bands agreed in step 0 (e.g. coverage within
  75–85 %); a candidate that fails one is rejected, however accurate.
- **Guardrails next:** metrics judged against the champion. A guardrail with a threshold
  agreed in step 0 fails when the candidate breaks it in either partition; one without fails
  when the candidate is shown worse than the champion in either partition (interval excludes
  0). A failure rejects. Record for each metric whether it is an acceptance check or a
  guardrail.
- **Decide on the primary metric** with the block-bootstrap interval of the gap:

| Selection partition | Verification partition | Decision |
|---|---|---|
| shown better, overall or in the target slice | same sign, no segment shown worse (the target slice counts as a segment) | **wins** |
| shown better | opposite sign, or a segment shown worse | **not adopted**: the gain depends on the partition; diagnose |
| neither shown better nor worse | any | **no difference**: the simpler model wins (fewer features, cheaper process) |
| shown worse | any | **rejected** |

- The selection partition chooses, the verification partition confirms; never re-select on
  verification. With several candidates, apply the rule unchanged to each.
- Report the target slice separately from the overall result, segments next to the total,
  every candidate of the round (losers included), and the run time and cost actually spent
  against the plan. Say plainly when a candidate won for another reason or missed its target.
- Run the project's standard diagnostics with every candidate, so that a failure shows its
  mechanism without a new analysis.

**9. Diagnose before the next idea** when a gap persists or a change fails: find the
mechanism, measure it on the slice, and derive the next candidate from it.

**10. The user promotes or rejects.**
- At most one promotion per decision. Other winners of a parallel round are re-tested on top
  of the new champion, because gains need not add up.
- After the decision, not before, add one short entry to the project's history: change,
  result, diagnosis, next step. Rejected candidates get one too; the history stops the same
  idea from being retried.
- Keep the code lean: delete the replaced champion's code (version control keeps it); a
  candidate close to the champion becomes a parameter, not new code. Keep stored full
  predictions only for the champion and the candidates still open.
- Back to step 7, or to the preparation step a finding changed (use case, data, deployment).

## Once per project: preparation (steps 0–6)

**0. Use case.** Ask before anything technical:
- the decision the predictions feed, who consumes them, timing, horizon, granularity;
- the cost of errors; the **primary metric** (its level and weighting) and the **guardrail
  metrics**, starting from the definitions already in use; comment where a metric does not
  fit the data or the decision; the point forecast that serves the decision (the mean when
  totals must be unbiased, with bias reported next to accuracy);
- required properties of the model (e.g. a smooth, monotonic price response for price
  optimisation; intervals; explanations; extrapolation beyond the training range by physical
  or domain laws), written as testable **acceptance checks**, which are gates;
- today's method with its documentation (the primary baseline), and the gain that would
  change a decision.

**1. Data, stack and infrastructure.** Read what is visible first, then ask in one grouped
round: data location and access, where analysis runs and who executes it, resources and their
cost structure, where results live and whether work is versioned, constraints. Ask how the
user weighs their own time against compute or cloud cost; turn the answer into one or two
working rules, confirm them and record them.

**Cycle length.** Estimate the time from a decided change to its result in front of the user
and say it. Beyond about 15–30 minutes, propose screening runs. Push back on choices that
weaken validity, naming the consequence.

**2. Explore the data:** the potential beyond simple rules and the main challenges, ending
in a short *potential and challenges* summary and the hand-over list in the reference.

**3. Design the evaluation** before any model:
- the same information as at decision time: drivers as they were forecast, publication lags,
  the incomplete current period;
- a split chosen from the data that reproduces deployment and respects dependence, with the
  evidence shown to the user; usually a rolling origin for forecasts;
- identical rows for every method; disjoint selection and verification partitions that each
  represent deployment (e.g. alternating months);
- significance by block bootstrap over the independent unit, and the detectable effect size;
- segments by information known at forecast time; a reproducibility check after refactors;
- where the use case depends on rare or extreme states, a slice for them (e.g. beyond the
  training range) with proper scores, as a guardrail.

**4. Non-ML baseline:** today's method reproduced from its documentation, and simple
rule-based benchmarks with explicit fallbacks, scored on the evaluation rows. The best one is
the reference throughout.

**5. Iteration 0:** the smallest model that meets the acceptance checks. Its run time is the
cycle length; it is the first champion.

**6. Exploratory shot:** one quick model that addresses all exploration findings at once,
time-boxed with the user's agreement. Never promoted, and its code is not reused: useful
features are rebuilt one per challenger. Same rows and information cutoff; analysed on the
selection partition only. Its output is a ranked list of candidates with hypothesis, target
slice, expected gain and cost.

## Results and reporting

- Define the champion and each candidate in one place; generate every table from that
  definition, so a promotion updates them on the next run.
- Results are readable without the code: state definitions, rules and decisions with their
  context where the results live. Write summaries after reading the outputs.
- Lead with what decides: acceptance checks, guardrails, the protocol result against the
  champion, the target slice, the standing against the baseline.
- Say plainly when a change missed its target, when a gain is not significant, or when an
  earlier claim turned out to be noise.
- End with the recommendation and the question the user must answer. Do not commit unless
  asked.
