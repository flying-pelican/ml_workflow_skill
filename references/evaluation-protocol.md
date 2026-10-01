# Evaluation protocol — details and rationale

Read this when setting up an evaluation, or when a result is close and the decision rule
needs care.

## Contents
1. Decision context and information cutoff
2. Benchmarks
3. Rolling origin and retraining
4. Selection and verification periods
5. Significance: block bootstrap of the WAPE gap
6. Segments and slices
7. Metrics
8. Canonical methods and how this protocol deviates

## 1. Decision context and information cutoff

Write down, before modelling (step 0 of the loop collects most of this from the user):
- the decision the forecast serves (e.g. "late today, decide how to allocate resources for
  tomorrow") — this fixes the **horizon** (here: 1 period) and often makes long horizons
  irrelevant;
- **when** the forecast is made and therefore the **information cutoff**. A forecast made
  before day T−1 is complete cannot use day T−1 of the target series: use data up to T−2.
  Exogenous inputs already observed at that time (e.g. day T−1's maximum temperature) may use
  T−1;
- which exogenous data would really exist at that time: archived *forecasts* of exogenous
  drivers (e.g. weather) instead of their observed values; publication lags of reference data
  (some official statistics appear about two years late — check their year-to-year stability
  before using a recent year);
- data issues that would distort evaluation: system outages (exclude), operational
  pseudo-entities (remove), left-censored attributes (e.g. "age" of entities older than the
  data is unknown, and counting from the data start turns it into a disguised time index).

## 2. Benchmarks

If the decision already relies on a method (a planner's rule, a spreadsheet, a legacy
forecast), reproduce it from its documentation — ask the user for it — and score it first; it
is the business reference. Then score simple history-only methods on exactly the evaluation
rows before any ML: naive (recent
mean), seasonal naive (same weekday last week), exponentially weighted mean, and a profile
(mean of the same weekday over the last four weeks). Keep their fallbacks for entities with
little history explicit (own recent mean, then nearest neighbours) and document them. The best
of these is the bar; its weaknesses (lag behind the season, blind to exogenous drivers) are
the model's opportunities.

## 3. Rolling origin and retraining

- Every operating day of an evaluation year is a target day; each is forecast as of its own
  cutoff.
- Retrain in blocks (month or week): the model for a block is trained on all rows up to two
  days before the block's first target day. Later days in a block use a slightly older model —
  conservative.
- The refit cadence is itself a model property. Monthly refits are cheap for feature
  exploration; weekly refits are closer to production and matter when new entities appear
  (a model that has never seen a new area extrapolates from its neighbours).
- Prefer a fixed model capacity (e.g. a fixed number of boosting rounds) over early stopping
  on a single late validation slice: in one case early stopping picked 52 trees for one
  forecast date and 2362 for another, and the small model shrank every entity towards the
  average.

## 4. Selection and verification periods

How to choose the split in general (groups, space, overlapping labels, distance matching) and
how to derive the two partitions from it: `validation-split-design.md`. For a rolling-origin
forecast evaluation:

Split the target days into two periods that each contain every regime of the year —
alternating months (Jan, Mar, May, … vs. Feb, Apr, Jun, …) worked. Choose on the selection
period; the verification period only confirms. A first-half/second-half split would compare a
winter–spring regime with a summer–autumn one and confuse seasonal effects with model effects.

Earlier variant (multi-week horizon): two forecast dates, one for selection, one for
verification. Same logic, far less data — prefer the rolling year when the use case allows.

## 5. Significance: block bootstrap of the metric gap

The gap is computed on the **primary metric agreed in step 0**; WAPE is used as the example
below. For challenger a and reference b on the same rows:

    gap = (Σ|y − ŷa| − Σ|y − ŷb|) / Σy        (negative: a is better)

Resample the **independent unit** found by the split design — here **calendar weeks** — with
replacement (all rows of a unit together; groups or spatial blocks in other designs), recompute
the gap,
take the 2.5 % and 97.5 % percentiles of 2000 draws. A gap whose interval excludes zero is
"shown". Compute per period × segment.

Why weeks (in a panel whose series share daily exogenous shocks): all series share a day's
exogenous and calendar shocks (e.g. weather), and such shocks persist for several days, so the effective sample size is closer to the number of weeks than to the
number of entity-days. Resampling entity-days or single days gives intervals that are far too
narrow. Sanity check: comparing a method with itself must give exactly 0 with a 0-width
interval.

Expect wide intervals: with one year (≈ 26 weeks per period), model-vs-model gaps of ~1.5
WAPE points and model-vs-baseline gaps of 2–3 points were needed for significance. Paired
comparisons between two similar models are tighter than comparisons with a baseline.

Algorithm (language-neutral):

    input: per row — actual y, forecast a, forecast b, block id (e.g. the week's Monday)
    1. per block k: Y[k] = Σ y,  A[k] = Σ |y − a|,  B[k] = Σ |y − b|      (K blocks)
    2. point estimate: gap = (Σ A − Σ B) / Σ Y
    3. repeat N = 2000 times:
         draw K block indices uniformly with replacement
         gap* = (Σ A[drawn] − Σ B[drawn]) / Σ Y[drawn]
    4. interval: 2.5 % and 97.5 % percentiles of the N values of gap*
    5. "shown better" if the upper bound < 0, "shown worse" if the lower bound > 0
    use a fixed random seed so results are reproducible

## 6. Segments and slices

- **History segments at forecast time**: new (no known history), short (1–28 days), medium
  (29–364), long (≥ 365). New and short behave completely differently from established
  entities; a model can halve the error on new entities and lose on short ones at once.
- **Diagnostic slices**: month, day type (workday / weekend-holiday), observed and forecast
  weather, entity volume quintiles, opening cohort, distance to known territory. Use them to
  localise a gap, not to select models (selection uses the protocol).

## 7. Metrics

The primary and guardrail metrics come from step 0 of the loop (the user's decision and
existing definitions). For other metrics the bootstrap works the same way: per block, keep the
sums the metric is built from (e.g. squared errors and counts for RMSE, pinball losses for
quantiles, costs for a cost-weighted error), resample blocks, and recompute the metric of each
model and their difference per draw. Defaults that worked for demand forecasting:

- WAPE (Σ|y − ŷ| / Σy) as the headline: scale-free across entities, robust to zeros.
- Bias ((Σŷ − Σy) / Σy) alongside — a lower bias with a higher WAPE is a common sign of a
  feature that corrects the level on average but adds variance (e.g. a seasonal proxy).
- Mean absolute error in target units, and the share of entity-days where each forecast is
  closer, for the business reading.
- Feature importance: split counts (biased towards high-cardinality features), the model's
  own importance, and mean |SHAP| in target units; read them together.

## 8. Canonical methods and how this protocol deviates

- **Rolling-origin evaluation** (Tashman 2000, *Out-of-sample tests of forecasting
  accuracy*, IJF): used as is, with a fixed horizon of one day and retraining blocks.
- **Block bootstrap for dependent data**: non-overlapping blocks (Hall 1985, Carlstein 1986)
  or moving blocks (Künsch 1989). Deviation: blocks are *calendar weeks* — fixed,
  non-overlapping, aligned to the calendar because shocks are shared across all series by
  date — and the statistic is a paired WAPE difference over a panel, not a single series.
- **Diebold–Mariano test** is the classical test for equal predictive accuracy of two
  forecasts. It was not used here because the loss is aggregated over a panel (WAPE) and the
  cross-sectional dependence is strong; the week bootstrap handles both directly. If a single
  series is evaluated, Diebold–Mariano with a HAC variance is the standard alternative.
- **Champion/challenger** is industry practice (model risk management, A/B deployment), not a
  formal method; the rules in SKILL.md are this workflow's own and should be stated to the
  user as such.
