# Diagnosis and pitfalls

Read this when a challenger fails, when a gap to the champion or the baseline persists, or
before adding a feature "because it might help".

## Contents
1. How to diagnose a gap
2. Worked examples (daily demand per entity, one year of history)
3. Leakage and information pitfalls
4. Engineering pitfalls

## 1. How to diagnose a gap

Work from the error, not from candidate features:

1. **Localise.** Compute the excess absolute error (challenger or champion minus the
   reference) per month, segment, entity, cohort, and day. Ask what share of the total excess
   sits in the top slices. Gaps are usually concentrated: one month, one cohort, a few days.
2. **Look at the extreme cases.** List the worst days or entities with actuals, both forecasts,
   and the relevant inputs (observed and forecast weather, history, location).
3. **Explain one forecast.** Break it into feature contributions (e.g. SHAP values, or the
   model's own attribution method) and look at what pushed it. Retrain the exact model if
   needed (same data and seed should reproduce it; verify).
4. **Check the training data behind that contribution.** E.g. which training days sit near
   that feature value, and what happened on them.
5. **Name the mechanism, then test it cheaply** (a table or a split of the training data)
   before building the feature. The next challenger implements the mechanism.
6. Write the diagnosis into the challenger's description, so the reason for the change stays
   with the change.

Correlating residuals with candidate inputs is a quick screen: weak correlations (≈ 0.05–0.1)
mean the information is not missing; a strong one points at the mechanism.

## 2. Worked examples

All from one dataset: daily demand for a few hundred geographically clustered entities, about
one year of training history before the evaluation year, demand strongly tied to temperature
and the seasonal profile, new entities added in waves, and archived forecasts of the exogenous
drivers available.

**Shrinkage was capacity, not missing inputs.** A gradient-boosted model lost to a
same-weekday profile baseline on established entities. Lags barely correlated with its errors; early stopping had chosen 52
trees on one date and 2362 on another. A fixed 1000 trees fixed it. Lags then helped on one
date and hurt on the other — not adopted.

**Day of year replayed one past year.** With only one previous year of training data, day of
year (the season proxy) reproduced that year's seasonal shape including its unusual winter:
it corrected the profile's lag in autumn but over-corrected two months of late winter and
spring. Dropping it was a tie overall — fewer features won. Forecasts of the exogenous
drivers later did the season's job properly.

**A disguised time index.** Entity age counted from the data start for entities older than
the data made "age" a time index; the first model looked better for the wrong reason. Fix:
treat their age as unknown (cap value). The honest model scored worse — and was kept.

**An expansion into unseen territory.** 98 % of a segment's excess error sat in one month and
one cohort: about 40 entities opened in an area the monthly model (trained up to the end of
the previous month) had never seen. Latitude/longitude extrapolated the level of the nearest
busy area (+67 %). Not the ramp-up (all opening waves ramped alike), not the history length.
A distance-to-known-entities feature failed (in training, distance was confounded with
location); descriptors of the surroundings (jobs and residents nearby) did not carry over
either; **weekly refits** fixed most of it (WAPE 0.70 → 0.42 on the target slice).

**A feature standing in for missing ones.** A temperature forecast — strongly correlated
with the seasonal profile — gained in some months and lost in others. The worst days: dry
days just below freezing were forecast at almost zero demand, because the training days at
that temperature with collapsed demand were snowstorms; and at every temperature, dry days
ran 20–30 % above wet ones, so the learned temperature effect averaged over both. The feature
had absorbed the effects of co-occurring drivers the model could not see. Adding the
precipitation-probability forecast fixed it: significant gains in both partitions, no month
worse.

**A rare event cannot be measured.** A forecast of a rare event (snowfall) was the weakest
feature: few event days, many false alarms in the evaluation year, its effect learned from a
training year with more events. No difference shown — fewer features won.

## 3. Leakage and information pitfalls

- Incomplete current day at forecast time (use T−2 for the target series).
- Observed exogenous data in place of their forecasts (e.g. weather) → upper bound only.
- Publication lags of reference data (official population or employment statistics) — check
  stability across years.
- Left-censored attributes (age, first-seen dates) acting as time indices.
- Outage days and operational pseudo-entities inflating errors or teaching wrong effects;
  near-outages (e.g. a city-wide travel ban day that slipped past an outage threshold) can
  teach "cold means zero".
- Features computed with windows that silently include the target day; verify windows by a
  hand calculation for one entity-day.

## 4. Engineering pitfalls

- **Row order is not a key.** Attach predictions by key (entity, date) and compare forecasts by
  key; comparisons by row position raise false alarms after joins or reorderings.
- **Refactors must reproduce the champion exactly.** Check identity after every change to
  shared code; tiny floating-point differences from a reordered computation usually do not
  change the model, but verify.
- **One implementation of features, benchmarks, periods and the bootstrap**, shared by every
  analysis, so all of them score the same rows the same way.
- **Print the checks with the results** (identity of the champion, errors, row counts), so a
  long run cannot silently report numbers from a broken state.
