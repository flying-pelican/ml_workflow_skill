---
name: ml-model-iteration
description: Disciplined, incremental improvement of a predictive model (forecasting models on panel/time-series data, but also other supervised ML) through champion/challenger iterations with a pre-agreed, data-driven evaluation protocol. Use this skill whenever the user wants to improve, iterate on, tune, or add/drop features of a forecasting or ML model, compare a new model version against the current one or a baseline, decide whether a change is "really better", set up a backtest, cross-validation or train/validation/test split, choose between random, grouped, time-based or spatial validation, translate a business use case into model requirements (e.g. a smooth, monotonic price response for price optimisation), or diagnose why a model loses to a simple benchmark — even if they don't say "champion", "challenger" or "iteration". Also use it when the user asks how to evaluate or select between models, or wants a careful, step-by-step modelling workflow instead of a big one-shot model.
---

# ML model iteration: champion, challenger, protocol

A workflow for improving a model one verifiable step at a time. It exists because the usual
failure of model development is not a lack of ideas but believing in gains that are noise,
leakage, or specific to one period — and stacking further changes on top of them. Every step
here is designed so that the user can see what changed, why, and whether it is real.

The user decides every promotion. Present evidence and a recommendation; do not promote,
commit, or move on to the next iteration on your own.

## Overview: preparation once, then the iteration loop

Steps 0–5 are **preparation**, done once per project (and revisited only when the use case,
the data or the deployment changes). Steps 6–9 are the **iteration loop**, repeated for every
challenger. Do not start the loop before the preparation is complete: without an agreed use
case, metric, exploration, evaluation design and baseline, iteration results cannot be judged.

### Preparation (once)

0. **Understand the use case the predictions serve** (see *Step 0: the use case and what it
   requires of the model*). Which decision they feed, when, at what granularity, what errors
   cost — and which properties the model must have for that decision (e.g. a smooth, strictly
   monotonic price response for price optimisation). These requirements constrain the model
   class, the metric and the acceptance checks of every later iteration.
1. **Interview the user on data, stack and infrastructure** (see *Step 1: data, stack and
   infrastructure*). Everything after this step — how data is read, where features and the
   evaluation are implemented, how long runs are handled — is adapted to the answers. Comment
   critically on the choices, above all on the expected iteration cycle length (see *Comment
   on the choices: cycle length*).
2. **Explore the data to identify the potential and the main challenges** (see *Step 2:
   explore the data*): how much of the variation a model could explain beyond simple rules,
   and what will make it hard (cold starts, anomalies, short history, drivers not available at
   decision time, shifts). The evaluation design, the baseline and iteration 0 are derived from
   it; without it they are guesses.
3. **Design the evaluation** (before any model): the horizon, when predictions are made and
   what is known at that moment follow from step 0. **Choose the validation split from the
   data** — what is new at prediction time (time, entities, places), how the data depend on
   each other — and show the user the evidence (`references/validation-split-design.md`).
   Build the evaluation set to match. → see *What makes an evaluation valid* below and
   `references/evaluation-protocol.md`.
4. **Establish the non-ML baseline** (see *Step 4: the non-ML baseline*): the method the
   decision relies on today, if one exists — reproduced from its documentation — and simple
   rule-based benchmarks, all scored on the evaluation set. This is the bar every model has to
   clear, and the number the business case is measured against.
5. **Iteration 0**: the smallest sensible model that meets the step-0 requirements, chosen
   from the data exploration. It is expected to lose; its job is to give a floor and a working
   pipeline. Measure its run time: that is the cycle length every later iteration will pay.
   Iteration 0 is the first champion.

### Iteration loop (repeat for every challenger)

6. **One challenger at a time** against the current champion — one change, one hypothesis
   (see *How to cut iterations*).
7. **Run, check the step-0 requirements, then judge by the protocol** (see *When a challenger
   wins*). Report the target slice the change was aimed at separately from the overall result.
8. **If a gap persists or a change fails, diagnose before inventing the next feature**
   (see `references/diagnosis-and-pitfalls.md`). The next challenger should follow from the
   diagnosed mechanism, not from a list of plausible features.
9. **The user promotes or rejects.** The loser moves to the history; the results again show
   one champion and at most one challenger. Then back to step 6 — or, if a finding changes the
   use case, the data or the deployment, back to the relevant preparation step.

## Step 0: the use case and what it requires of the model

Ask before anything technical — the answers shape model choices that are expensive to change
later. Clarify, in the user's words:

- **The decision** the predictions feed and who or what consumes them: a person reading a
  number, a rule, a downstream optimiser, an automated system.
- **Timing and granularity**: when predictions are made, for which horizon, per what unit
  (entity, product, region, day, hour), how often they are refreshed.
- **Cost of errors**: symmetric or asymmetric (running out vs. overstocking), absolute or
  relative, which segments matter most. This picks the metric and the loss — possibly a
  quantile or a cost-weighted error instead of a plain average.
- **Metrics the predictions are judged by** — ask explicitly, and ask for the definitions
  already in use (reports, dashboards, today's method):
  - the **primary metric** that decides between champion and challenger (e.g. WAPE, MAE,
    RMSE, pinball loss for quantiles, a cost-weighted error, a business KPI computed from the
    predictions), and at which level it is computed — per entity and day, per segment, on
    totals — and how entities are weighted (by volume, equally, by value);
  - **guardrail metrics** that must not get worse (e.g. bias, interval coverage, error on a
    key segment, stability of predictions between retrains);
  - how the business reads the result (e.g. "error in units per day", "share of days off by
    more than 50 %"), so results can be reported in those terms as well.

  Comment critically where a metric does not fit the data or the decision, and say why: MAPE
  is undefined or explodes with zeros and small values; RMSE is dominated by the largest
  entities and outliers; an unweighted average over entities lets tiny entities decide; a
  point-error metric cannot judge intervals; a metric that differs from the training loss
  favours models trained for something else. Agree on one primary metric before the first
  evaluation — it is the one the protocol's gap and significance test are computed on.
- **Required properties of the model's behaviour**, for example:
  - *price or promotion optimisation* — demand must respond to price smoothly and strictly
    monotonically, otherwise the optimiser exploits wiggles of the fitted curve; this calls
    for monotonic constraints, smooth model classes (e.g. shape-constrained or additive models)
    or a parametric price-response layer, and a check of the response on a price grid;
  - *inventory or staffing* — prediction intervals or quantiles, not only a mean;
  - *explanations for users or an explanation agent* — per-prediction attributions in target
    units, stable across retrains;
  - *regulated or high-stakes use* — interpretability, documented features, fairness checks;
  - *operations* — latency, retrain budget, robustness to missing inputs, consistency of
    predictions across aggregation levels.
- **How the decision is made today**: is there an existing method — a planner's rule, a
  spreadsheet, last year's value plus a factor, a vendor or legacy forecast? If so, ask for its
  documentation (definition, inputs, how it treats new or sparse entities, where its outputs
  are stored, known weaknesses) or for its historical outputs. It becomes the primary baseline
  in step 4.
- **What "good enough" means**: the gain over today's method that would change a decision.

Write the requirements down as **acceptance checks** (e.g. "price response monotonic and
smooth for every product on the observed price range", "90 % interval coverage within 85–95 %",
"bias within ±5 %"), and record the primary and guardrail metrics next to them.
They are gates: a challenger that fails one is not compared on accuracy at all. Comment if a
requirement and the user's other choices conflict (e.g. a strict monotonic price response with
a model class that cannot be constrained).

## Step 1: data, stack and infrastructure

This workflow makes no assumption about tools. Before the first plan, find out how the user
works — first from what is already visible (repository files, project notes, earlier
conversation), then by asking only what is still open. Ask in one short round, grouped, with
examples so the user can answer in a few words, e.g.:

- **Data location and access** — a local dump (CSV, Parquet, Excel), a database or warehouse
  (SQL), Delta tables on Databricks, files in cloud object storage? How large, how is access
  granted, may data leave its environment?
- **Where analysis runs** — local Jupyter notebooks or scripts, a remote notebook platform
  (e.g. Databricks), a cluster or cloud jobs? Who executes: you directly, or the user (e.g.
  you write notebook cells, the user runs them)?
- **Language and libraries** — Python (pandas, polars, PySpark), R, SQL; model libraries
  already in use (scikit-learn, LightGBM, XGBoost, CatBoost, Spark ML, …).
- **Run budget** — how long may one evaluation run take, and are there compute costs (cluster
  time) that should shape the number of refits or bootstrap draws?
- **Results and history** — where results should live (notebooks, a report, an experiment
  tracker such as MLflow), whether work is versioned (git), and whether anything should be
  committed or published without asking (default: no).
- **Constraints** — data protection, licences that forbid redistributing the data, company
  standards for code or environments.

Record the answers where the project keeps its notes (or tell the user they should be noted),
so the next session does not ask again. Then adapt:

- implement the shared logic (features, benchmarks, split, bootstrap) where the stack shares
  code — a module or package, a notebook included by the others, SQL views or tables;
- push heavy work to where the data lives (e.g. aggregate in SQL or Spark before pulling
  results; run long refit loops as jobs) instead of moving large data;
- size the evaluation to the run budget (refit cadence, number of draws), and say what the
  budget costs in statistical power.

## Comment on the choices: cycle length

Do not just take the setup as given. After the interview, give the user short, concrete,
critical feedback on their choices and their consequences, and propose alternatives — the
user decides. Above all, estimate the **iteration cycle length**: the time from deciding on a
change to having the protocol result in front of the user (data preparation + training of all
refits + scoring + bootstrap + review). The cycle length decides how many hypotheses can be
tested in a working session, and a slow loop quietly pushes towards bundling several changes
into one iteration — which makes results uninterpretable.

- **Estimate it early and say it out loud.** E.g. "one full evaluation is 12 monthly refits ×
  ~10 s + bootstrap ≈ 5 min; weekly refits make it ≈ 13 min". Remote platforms add cluster
  start-up and queueing; user-executed notebooks add a round trip per run.
- **Too long (beyond roughly 15–30 minutes per iteration)?** Propose a two-tier setup:
  *screening* runs on a cheaper configuration (coarser refit cadence, a stratified subsample of
  entities, fewer or larger-step boosting rounds, features computed once and cached, fewer
  bootstrap draws), and the *full protocol* only for candidates that might be promoted. Push
  heavy computation to where the data lives, run refits in parallel or as jobs. Say what each
  shortcut costs (e.g. a subsample widens the intervals; a coarser cadence hides effects of new
  entities).
- **Very short and cheap?** Then the risk is the opposite: many quick challengers against the
  same selection partition inflate false wins (each comparison is a new chance to fit noise).
  Keep the verification partition untouched by tuning, keep the iteration history complete
  (including rejected ideas), and consider a final holdout for the end of the project.
- **Push back on choices that weaken validity**, naming the expected consequence: a random
  split for data with time or spatial dependence (optimistic errors, overrated location
  features), observed instead of forecast exogenous drivers (an upper bound, not a forecast),
  a single evaluation date or a too short evaluation period (one regime, no significance),
  several changes in one iteration "to save time" (no attribution), early stopping on one late
  slice (unstable model size).

## Step 2: explore the data

Explore before choosing the evaluation, the baseline or any feature. The purpose is to
identify **the potential** — how much of the variation is explainable at all, and how much of
it simple rules already capture (e.g. explained variance of drivers and calendar on the total;
the share of entity level vs. shared time effect; where simple rules will fail) — and **the
main challenges** — cold starts, anomalies, short history, drivers that are not available at
decision time, regime shifts, dependence that complicates evaluation. End the report with a
short *potential and challenges* summary the user can react to. The report should be
reproducible (in whatever tool the stack uses), and its findings are referenced later as the
reason for design choices. Cover at least:

- **Data quality and scope** — coverage, missing periods, duplicates, outages or system-wide
  anomalies (flag and exclude rather than learn them), pseudo-entities that are not real
  demand (test, depot or internal accounts), units and time zones.
- **Entity lifecycle** — when entities appear and disappear, waves of additions, history
  length at any given time, left-censoring at the start of the data.
- **Target** — distribution, share of zeros, scale differences between entities, trend.
- **Drivers** — calendar (weekday, holidays — check each holiday's effect, they differ a lot),
  season, exogenous drivers (e.g. weather): how much of the variation each explains, and
  whether they are available *as forecasts* at decision time (compare forecast against
  observed).
- **Variance decomposition** — how much is entity level, how much shared time effect, how much
  residual; this tells whether location/entity features or time features matter most.
- **Dependence structure** — autocorrelation, shared shocks across entities, spatial
  clustering; this feeds the split design (`references/validation-split-design.md`).
- **New entities** — ramp-up after opening, and what is known about them in advance
  (location, surroundings), since cold starts are often where models and benchmarks differ.

Details and checks: `references/data-exploration.md`.

## Step 4: the non-ML baseline

Establish the baseline before iteration 0, on exactly the evaluation rows:

- **Today's method first.** If step 0 found one, reproduce it from its documentation and
  check the reproduction against its historical outputs where available. If documentation is
  missing or ambiguous, ask the user rather than guessing — a baseline that is weaker than the
  real method inflates every later gain. If only its outputs exist, score those.
- **Simple rule-based benchmarks** in any case, because they show what plain history already
  achieves: last value or recent mean, the same period last cycle (e.g. same weekday last
  week), an exponentially weighted mean, and a seasonal profile (e.g. the mean of the same
  weekday over the last weeks). Make fallbacks for entities with little or no history explicit
  (own recent mean, then nearby or similar entities) and document them.
- **Report** each benchmark overall and per segment and month, and name its weaknesses (e.g.
  it lags the season, it is blind to exogenous drivers, it fails on new entities). These
  weaknesses are where a model can add value; where the baseline is already hard to beat, say
  so early.
- The best simple benchmark, or today's method if it is better, is the **reference** that the
  champion is compared against throughout.

## How to cut iterations

- **One change per iteration.** A change is one feature, one coherent feature group that
  tests one hypothesis (e.g. forecasts of two related exogenous drivers when the diagnosis
  says the model lacks that driver), one removal, or one training/process change (refit
  cadence, loss, tree count) with the feature set unchanged. Two unrelated changes in one challenger make the
  result uninterpretable.
- **State the hypothesis and the target slice before running.** E.g. "with only one year of
  history, dropping the day-of-year feature should stop it from replaying that year's
  seasonal shape in late winter", or "weekly refits should fix the entities that opened in an
  area without training coverage". After the run, check that slice explicitly. A challenger
  can win for a different reason than intended — say so plainly instead of claiming the target was hit.
- **Removals are iterations too.** When a feature contributes little, a challenger without it
  tests whether it can go; ties favour the smaller model.
- **Start from the benchmark you must beat.** Giving the model the strong simple baseline as a
  feature (e.g. a seasonal profile such as the same-weekday mean of recent weeks) makes it
  learn corrections instead of re-learning levels.
  Replicate the baseline exactly (verify equality on the evaluation rows) including its
  fallbacks, so missing values do not silently differ.
- **Keep the history short and factual.** Each retired iteration gets one entry: what it
  added/removed, overall score, the protocol result, and why it lost or was replaced. This is
  the institutional memory that stops the same idea from being retried.
- **Keep iterations fast by default.** Prefer the cheaper setup (e.g. monthly refits) while
  exploring features; move to the expensive realistic setup (e.g. weekly refits) as its own
  iteration when a diagnosis points at it.

## When a challenger wins

First check the step-0 acceptance checks (e.g. monotonic, smooth price response; interval
coverage). A challenger that fails one is rejected, however accurate it is. Then compare
challenger and champion on the same rows with the agreed protocol (selection and
verification partitions — time periods, groups or spatial blocks, whatever the split design
chose — and significance by block bootstrap; see next section). Then:

| Selection partition | Verification partition | Decision |
|---|---|---|
| better, interval excludes 0 (overall or in the target slice) | same sign, no segment significantly worse | **wins** |
| better | better | **wins** |
| no difference shown | no difference shown | **fewer features / simpler process wins** |
| better | worse, or no gain at all | **not adopted** — the gain is partition-dependent; diagnose it |
| no gain | better | **not adopted** — the selection partition decides; diagnose why the partitions differ |

- The **selection partition chooses, the verification partition confirms**. Never re-select on
  the verification partition; if verification contradicts, the change is not adopted, however
  good the overall number looks. A large gain in one partition with nothing in the other
  usually means the change is regime-dependent (see "A feature standing in for missing ones"
  in `references/diagnosis-and-pitfalls.md`).
- **Report segments, not only the total.** A challenger that wins overall but is significantly
  worse on a segment is a finding to show the user, not to bury.
- **Tie-break on simplicity only between equals.** When no difference is shown, the model with
  fewer features (or the cheaper process) wins. Process changes with identical features have
  no feature-count tie-break; weigh their cost explicitly.
- **Compare against the baseline too.** The champion's standing against the simple benchmark
  is the number that matters for the business case; keep it visible and updated.

## What makes an evaluation valid

A valid evaluation reproduces the decision situation and can tell a real gain from noise:

1. **Same information as at decision time.** Every feature of a target day must be computable
   from data available when the prediction is made — including the incomplete current period
   (e.g. a forecast made late on day T−1 for day T knows the target only up to T−2),
   publication lags of reference data, and exogenous drivers as they were *forecast* at the
   time (archived forecasts, not observed values — observed drivers only give an upper bound).
2. **A split that reproduces deployment and respects dependence, chosen from the data.**
   Measure what is new at prediction time and how far prediction points are from the training
   data (in time, entities, space, feature space), and how residuals depend on each other
   (autocorrelation, shared shocks, spatial range, groups). Then pick among random, stratified,
   grouped, time-based (holdout, rolling origin, hv-block), purged/embargoed, spatial-block and
   distance-matching splits — or a combination — and check the choice against the data.
   Method table, diagnostics and acceptance checks: `references/validation-split-design.md`.
   For forecasting this is usually a **rolling origin** over a long, representative period
   (Tashman 2000), retrained on the production schedule.
3. **Identical rows for every method.** All models and benchmarks are scored on exactly the
   same target rows and segments (store the benchmark rows once and join to them).
4. **Disjoint selection and verification partitions that each represent deployment** —
   e.g. alternating months rather than first vs. second half (so a seasonal feature is not
   chosen in one regime and checked in another), whole groups assigned at random, or
   interleaved spatial blocks.
5. **Significance from a block bootstrap over the independent unit** — the unit the
   dependence diagnostics identified: calendar weeks when all series share a day's shocks and
   shocks persist, spatial blocks, or groups. Smaller units understate the uncertainty. Report
   the gap challenger − champion (e.g. in WAPE) with a 95 % interval; the algorithm is
   specified step by step in `references/evaluation-protocol.md` and fits in a few lines in
   any language.
6. **Know the detectable effect size.** With one year, gaps of ~1.5 WAPE points (model vs.
   model) or 2–3 points (model vs. baseline) were needed to show anything. Do not chase tenths
   of a point; say when a difference cannot be resolved.
7. **Segments by information at forecast time**, e.g. history length (new / short / medium /
   long) measured at the forecast origin, plus calendar and exogenous-driver slices (e.g.
   month, weather) for diagnosis.
8. **Reproducibility checks.** After any refactor, verify the champion's forecasts are
   identical, compared by key (entity, date) rather than by row position. Store outputs in a
   deterministic order.

Details, rationale and the canonical methods behind each point: `references/evaluation-protocol.md`.

## Results layout (keeps results updatable)

Use whatever the project already uses for analysis (notebook, report script, dashboard,
document); the structure matters, not the tool.

- Intro: use case and information cutoff; **champion** feature table (feature → why);
  training setup; the evaluation protocol in a few bullets; **earlier iterations** (one entry
  each).
- One place that names the champion and defines each iteration (name → feature list, training
  settings). All champion tables and plots are generated from that definition, never typed as
  text, so promoting a challenger updates them on the next run.
- Champion vs. baseline section: overall table, significance by partition × segment (table +
  interval plot), by segment, by month (WAPE and bias), by day type and exogenous drivers, by entity volume,
  totals over time, feature importance.
- Challenger section (when one exists): what changes and the hypothesis; results by segment
  and month next to the champion; protocol tables against champion and baseline; the target
  slice; importance; a short summary written *after* reading the outputs.
- Feature importance with more than one measure — split or usage counts mislead (features
  with many distinct values get more splits); also show the model's own importance and mean
  absolute attributions (e.g. SHAP) in target units where the model type supports them.
- Feature, benchmark, split and bootstrap logic is implemented once and shared, so every
  analysis scores the same rows the same way.

## Reporting to the user

- Lead with the decision-relevant result: protocol outcome against the champion, the target
  slice, and the standing against the baseline. Tables over prose; numbers from the outputs.
- Say plainly when a change missed its target, when a gain is not significant, or when an
  earlier claim turned out to be noise.
- End with the recommendation and the question the user needs to answer (promote? next
  challenger?). Do not commit unless asked.
