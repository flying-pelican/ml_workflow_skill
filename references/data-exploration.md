# Data exploration before modelling

Read this at step 2 of the loop. The exploration is not a warm-up: each finding should end up
as a reason for a design choice (what to exclude, which split, which benchmark, which features
for iteration 0). Write it as a short report in the user's tool of choice, with one section per
question below and a one-line conclusion under each table or plot.

## Contents
1. Data quality and scope
2. Entity lifecycle
3. Target
4. Drivers: calendar, season, exogenous
5. Variance decomposition
6. Dependence structure
7. New entities and what is known about them
8. What the exploration must hand over

## 1. Data quality and scope

- Coverage per entity and period; gaps; duplicated keys; units; time zones and day
  boundaries.
- **System-wide anomalies** (outages, data-delivery failures): flag days whose total falls far
  below a robust local reference, e.g. below 10 % of the centred 15-day median of the total.
  Exclude them from training and evaluation instead of letting a model learn them. Check the
  near-misses by hand: a partial shutdown just above the threshold can still teach a model a
  wrong effect.
- **Pseudo-entities** that are not real demand (test, depot, workshop, internal accounts):
  find them by names, extreme values or odd patterns, and remove them everywhere.
- Zero rows: real zero demand or "not operating"? Decide how an entity's operating span is
  defined (e.g. first to last observed activity, zeros in between).

## 2. Entity lifecycle

- When do entities appear and disappear? Plot active entities and additions per week; look
  for waves.
- History length available at any date — this becomes the segment definition (new / short /
  medium / long history at forecast time).
- **Left-censoring**: entities older than the data have an unknown age. Do not count age from
  the data start (it turns "age" into a time index); treat it as unknown or capped.

## 3. Target

- Distribution, share of zeros, scale range across entities (affects the metric: a scale-free
  aggregate such as WAPE is robust to zeros and mixes entities fairly).
- Trend and level shifts of the total — but measure them on a fixed set of entities that
  existed throughout, otherwise additions of entities masquerade as trend.

## 4. Drivers: calendar, season, exogenous

- **Explained variation of the total** (on a fixed entity set, outages excluded): calendar
  only, exogenous only, both plus trend — linear and with a flexible model under
  cross-validation. Example result: calendar alone R² 0.07, weather alone 0.71, everything
  with a GBM 0.90 → the exogenous driver dominates the day-to-day variation.
- **Season vs. exogenous driver**: if the season is mostly a proxy for an exogenous driver
  (e.g. temperature), plan to model the driver, not the calendar — with one year of history a
  calendar season feature replays that single year.
- **Holidays**: compare each holiday with the same weekday one week before and after; effects
  differ widely (one holiday at 0.2× normal, another at 0.8×), so one binary flag is coarse;
  note which calendar the flag uses (observed vs. actual dates).
- **Availability at decision time**: for every exogenous driver, find out whether archived
  forecasts exist for the evaluation period, and compare them with the observations (error,
  correlation, calibration of probabilities; what share of the observed effect the forecast
  retains — e.g. R² 0.88 with forecasts vs. 0.90 with observations). Observations alone only
  give an upper bound.

## 5. Variance decomposition

Two-way decomposition of the (log) target over entities that exist throughout one year:
share of variance from the entity effect, from the shared date effect, and residual. Example:
entity 49 %, date 21 %, residual 30 %. A large entity share means level and location features
matter, and that cross-learning across entities needs a way to tell entities apart.

## 6. Dependence structure

- Residual autocorrelation over time (per entity and of the cross-sectional mean).
- Correlation of residuals across entities on the same date (shared shocks).
- Spatial clustering of residuals (correlogram / variogram) if entities have locations.
- Within-group correlation for hierarchies (stores in regions, patients in hospitals).

These numbers decide the validation split and the bootstrap unit
(`validation-split-design.md`).

## 7. New entities and what is known about them

- **Ramp-up**: mean target by weeks since opening for a cohort of additions, against
  established entities over the same calendar weeks (so season does not masquerade as
  ramp-up). Typical: the first days are partial and low, then a few weeks of growth.
- **What is known in advance**: location, planned opening date, surroundings (e.g. jobs and
  residents nearby from official statistics). Check how stable such reference data are across
  years (a correlation of ~0.99 between years makes the publication lag irrelevant) and how
  much of the entity level they explain compared with location alone — and test the hard case:
  trained on established entities, does it predict a new area?

## 8. What the exploration must hand over

- a **potential and challenges** summary, a few lines each:
  - *potential*: how much variation drivers and calendar explain beyond simple history (e.g.
    exogenous drivers explain far more of the day-to-day variation than the calendar, which a
    history-only rule cannot use), which segments simple rules will serve badly (new entities,
    regime changes), and a rough sense of the achievable gain;
  - *challenges*: cold starts and expansions, anomalies, short training history (one year
    means one example of each season), drivers available only as forecasts, dependence that
    makes the evaluation harder, data that cannot leave its environment;
- the exclusion rules (anomalous days, pseudo-entities) and the cleaned panel;
- the segment definition and the entity lifecycle facts;
- the dominant drivers and whether each is available at decision time;
- the dependence measurements for the split design;
- a short list of candidate features for iteration 0, each with the finding that motivates it.
