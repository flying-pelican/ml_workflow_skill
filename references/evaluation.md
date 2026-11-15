# Evaluation: information cutoff, split, partitions, significance

Read at step 3, and when a result is close and the decision rule needs care. `SKILL.md` holds
the binding rules; this file holds the choices behind them.

## Information cutoff

- A forecast made before day T−1 is complete cannot use day T−1 of the target series: use
  data up to T−2. Exogenous inputs already observed at that time may use T−1.
- Exogenous drivers as they were *forecast* at the time (archived forecasts), not observed;
  observed drivers give an upper bound only.
- Reference data with publication lags (some official statistics appear years late): check
  their stability across years before using a recent one; a correlation of about 0.99 between
  years makes the lag irrelevant.
- Left-censored attributes (the age of entities older than the data) counted from the data
  start become a disguised time index; treat them as unknown or capped.

## Choosing the split from the data

Reproduce what is new for the model at prediction time (periods, entities, places) and how far
prediction points are from the training data, while keeping dependent observations on the same
side. Ignoring dependence underestimates the error seriously (Roberts et al. 2017); blocking
too aggressively forces extrapolation the deployed model never faces and makes every model look
worse than it is. Show the user the evidence and record the choice with the results.

1. **Describe deployment, with numbers:** new in time (horizon, retrain cadence, cutoff); the
   share of prediction rows from entities with no or little history; the distribution of
   nearest-neighbour distances from prediction to training points, in space or feature space;
   whether labels span windows that overlap.
2. **Measure dependence on baseline residuals:** autocorrelation per series and of the
   cross-sectional mean, correlation across entities on the same date, the spatial range,
   intraclass correlation of groups, and adversarial validation between candidate splits.
   These ranges set block length, gap or buffer, and the bootstrap unit.
3. **Pick the method family:** random or stratified k-fold only for independent rows; group,
   time (rolling origin, hv-block), purged or embargoed, spatial-block or distance-matching
   splits otherwise, often combined (rolling origin with entity groups). k-fold on lagged
   features is valid only for purely autoregressive models with uncorrelated residuals
   (Bergmeir et al. 2018).
4. **Check the chosen split against the data:**
   - test → training distances resemble prediction → training distances (rolling origin with
     the production cadence reproduces the time distance automatically);
   - the share of new and short-history entities, regimes and key segments in the test folds
     resembles deployment;
   - residual correlation across the split boundary is negligible, or a gap removes it;
   - at least about 20 independent units per partition, otherwise say before running that
     intervals will be wide;
   - the baseline's error is stable on a stricter split; if it jumps, the looser split leaks.

**What a wrong split hid, in one case:** entities added in an area without training coverage
showed as a large error in one month, because the rolling origin reproduced the novelty in
time. A random entity-day k-fold would have hidden it and rated location features far too
highly: entity-level R² 0.69 for latitude and longitude with random folds, while location
over-predicted the new area's level by 155 %.

## Rolling origin and retraining

- Retrain in blocks (month or week): the model for a block is trained on all rows up to the
  cutoff before the block's first target day.
- The refit cadence is a model property; weekly refits matter when new entities appear.
- Prefer a fixed model capacity over early stopping on one late validation slice: in one case
  early stopping picked 52 trees for one forecast date and 2362 for another, and the small
  model shrank every entity towards the average.

## Selection and verification partitions

Two disjoint partitions that each represent deployment: alternating time blocks (odd vs. even
months, so both contain every season; first half vs. second half would confuse seasonal with
model effects), whole groups at random stratified by size, or interleaved spatial blocks. Tune
inside the selection partition only (nested); optionally keep a final untouched test set for
the end of the project.

## Significance: block bootstrap of the gap

The gap is computed on the primary metric agreed in step 0, paired on the same rows. Resample
the **independent unit** found by the split design. In a panel whose series share a day's
exogenous shocks and where shocks persist, the effective sample size is closer to the number
of weeks than of entity-days: resample **calendar weeks**. Smaller units give intervals that
are far too narrow.

Per block, keep the sums the metric is built from (for WAPE: Σ y, Σ |y − a|, Σ |y − b|);
draw the blocks with replacement 2000 times, recompute the gap from the drawn sums, and take
the 2.5 % and 97.5 % percentiles. Use a fixed seed; a method compared with itself must give
exactly 0 with zero width.

Other metrics work the same way (squared errors and counts, pinball losses, costs). Expect wide
intervals: with one year (about 26 weeks per partition), model-vs-model gaps of about 1.5 WAPE
points and model-vs-baseline gaps of 2–3 points were needed. Paired comparisons of similar
models are tighter than comparisons with a baseline.

## Segments and metrics

- **History segments at forecast time:** new (no history), short (1–28 days), medium
  (29–364), long (≥ 365). New and short behave differently; a model can halve the error on new
  entities and lose on short ones at once.
- **Diagnostic slices** (month, day type, observed and forecast drivers, volume quintiles,
  cohort, distance to known territory) localise a gap; they do not select models.
- **Bias next to accuracy.** A lower bias with a higher absolute error usually means a feature
  corrects the level on average but adds variance (e.g. a seasonal proxy). When totals feed
  the decision, the point forecast is the mean, not the median that minimises absolute error.

## Canonical methods and the deviations

- Rolling-origin evaluation (Tashman 2000): used as is.
- Block bootstrap (Carlstein 1986; Künsch 1989). Deviation: fixed, non-overlapping calendar
  weeks, because shocks are shared across series by date; the statistic is a paired gap over a
  panel, not a single series.
- Diebold–Mariano is the classical test of equal accuracy; not used for panels with strong
  cross-sectional dependence and an aggregated loss, which the week bootstrap handles directly.
  For a single series, Diebold–Mariano with a HAC variance is the standard alternative.
- Champion/challenger is industry practice, not a formal method: the rules in `SKILL.md` are
  this workflow's own; state them to the user as such.
