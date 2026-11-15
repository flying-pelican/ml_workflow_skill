# Diagnosis and pitfalls

Read when a challenger fails, when a gap to the champion or the baseline persists, or before
adding a feature "because it might help".

## How to diagnose a gap

Work from the error, not from candidate features:

1. **Localise.** The excess error per month, segment, entity, cohort and day; what share of
   the total excess sits in the top slices. Gaps are usually concentrated.
2. **Look at the extreme cases** with actuals, both forecasts and the relevant inputs.
3. **Explain one forecast** by its feature contributions; retrain the exact model if needed
   (same data and seed must reproduce it).
4. **Check the training data behind that contribution:** which training rows sit near that
   feature value, and what happened on them.
5. **Name the mechanism, then test it cheaply** (a table, a split of the training data) before
   building anything. The next challenger implements the mechanism, and its description keeps
   the diagnosis.

Correlating residuals with candidate inputs is a quick screen: weak correlations (about
0.05–0.1) mean the information is not missing.

## Worked examples

From one panel of daily demand per entity with about one year of training history, a strong
exogenous driver and new entities added in waves.

**Shrinkage was capacity, not missing inputs.** A boosted model lost to a same-weekday profile
on established entities. Lags barely correlated with its errors; early stopping had chosen 52
trees on one date and 2362 on another. A fixed 1000 trees fixed it. Lags then helped on one
date and hurt on the other: not adopted.

**Day of year replayed one past year.** With one previous year of training, day of year
reproduced that year's seasonal shape, including its unusual winter: it corrected the
profile's lag in autumn but over-corrected late winter and spring. Dropping it was a tie, so
fewer features won. Forecasts of the exogenous driver later did the season's job properly.

**A disguised time index.** Age counted from the data start, for entities older than the data,
made "age" a time index; the first model looked better for the wrong reason. Treating their
age as unknown scored worse, and was kept.

**An expansion into unseen territory.** 98 % of a segment's excess error sat in one month and
one cohort: about 40 entities opened in an area the monthly model had never seen. Latitude and
longitude extrapolated the level of the nearest busy area (+67 %). Not the ramp-up, not the
history length. A distance-to-known-entities feature failed (confounded with location in
training), descriptors of the surroundings did not carry over; weekly refits fixed most of it
(WAPE 0.70 → 0.42 on the target slice).

**A feature standing in for missing ones.** A forecast of the main exogenous driver, strongly
correlated with the seasonal profile, gained in some months and lost in others. The worst days
were forecast near zero because the training days at that driver value had collapsed demand
for a different reason, a rare event the model could not see; and at every value, rows with a
second, unseen condition ran 20–30 % apart. The feature had absorbed the effects of
co-occurring drivers. Adding a forecast of the second driver fixed it: shown better in both
partitions, no month worse.

**A rare event cannot be measured.** A forecast of a rare event was the weakest feature: few
event days, many false alarms, its effect learned from a training year with more events. No
difference shown, so fewer features won.

## Leakage that is easy to miss

- Near-outages that slipped past the anomaly threshold can teach a wrong effect (a partial
  shutdown teaching "this condition means zero").
- Feature windows that silently include the target day: verify windows by a hand calculation
  for one entity-day.

## Engineering pitfalls

- **Row order is not a key.** Attach and compare forecasts by key (entity, date), never by row
  position.
- **Refactors reproduce the champion exactly.** Check identity after every change to shared
  code.
- **Print the checks with the results** (identity of the champion, row counts, missing
  forecasts), so a long run cannot silently report numbers from a broken state.
