# Choosing the validation split from the data

Read this before the first evaluation of a project, and again whenever the deployment target
changes (new entities, new regions, a longer horizon). The split decides what "better" means;
a split that does not match deployment can make a worse model win.

## Contents
1. The principle
2. Step 1 — describe the deployment target
3. Step 2 — measure the dependence structure
4. Step 3 — candidate methods and when each is valid
5. Step 4 — check the chosen split against the data
6. Step 5 — derive selection and verification partitions
7. Worked example (next-day demand per entity)
8. References

## 1. The principle

Choose the split that reproduces, as closely as the data allow, (a) **what is new** for the
model at prediction time and (b) **how far** prediction points are from the training data —
while keeping dependent observations on the same side of the split. Random splits are only
valid when neither time, space, group membership nor overlapping labels connect training and
test rows. Ignoring such dependence is known to underestimate the prediction error seriously
(Roberts et al. 2017). The opposite error exists as well: blocking too aggressively forces
extrapolation the deployed model never faces and makes every model look worse than it is.

Make the choice explicit, show the user the evidence for it, and record it with the results.

## 2. Step 1 — describe the deployment target

Answer from the use case and the data, with numbers where possible:

- **New in time?** Will predictions be made for periods after the training data (forecasting)?
  What horizon, how often is the model retrained, what is the information cutoff?
- **New entities?** What share of prediction rows belongs to entities (customers, stores,
  sites, patients) that have no or little training history? (In the worked example below:
  0.2 % of entity-days had no history, 2.4 % under four weeks.)
- **New places or feature regions?** How far are prediction points from training points in
  space or in feature space? Compute the distribution of nearest-neighbour distances
  prediction → training (geographic distance, or a standardised feature-space distance).
- **Overlapping labels?** Does a target span a window (multi-step totals, returns over
  horizons, events with durations) so that neighbouring rows share label information?

## 3. Step 2 — measure the dependence structure

Fit a quick reference model (or use the baseline) and inspect its **residuals**:

- **Temporal**: autocorrelation function of residuals per series and of the cross-sectional
  daily mean. The lag where it falls to noise level is the minimum block length / gap.
- **Cross-sectional shocks**: correlation of residuals between entities on the same date
  (shared weather, holidays, market moves). Strong → rows of the same date belong together.
- **Spatial**: correlogram or semivariogram of residuals (or Moran's I); the range where
  correlation vanishes is the minimum spatial block size or buffer.
- **Group / hierarchical**: share of residual variance between groups (intraclass
  correlation). Substantial → keep groups together.
- **Covariate shift between candidate splits**: train a classifier to tell candidate
  validation rows from training rows ("adversarial validation"); AUC near 0.5 means similar
  distributions, high AUC means the split tests extrapolation — compare with what deployment
  looks like (step 1).

Roberts et al. (2017) recommend blocking wherever dependence structures exist in the data,
even if the residuals of the fitted model look uncorrelated.

## 4. Step 3 — candidate methods and when each is valid

| Method | Valid when | Key rule |
|---|---|---|
| Random holdout / k-fold | rows independent; deployment interpolates within the same population | check residual dependence first; usually wrong for time series, panels, spatial data |
| Stratified k-fold | as above, with rare classes or segments | stratify on the target or segment so every fold contains them |
| Group k-fold / leave-one-group-out | deployment to unseen entities, or strong within-group correlation | all rows of a group in the same fold |
| Holdout of the last period | forecasting; enough data for one long test period | test period after training period; single estimate, high variance |
| Rolling origin / time-series CV (Tashman 2000) | forecasting with retraining | each test block is forecast from data before its origin; fixed horizon; many origins |
| k-fold on lagged features (Bergmeir, Hyndman & Koo 2018) | purely autoregressive models whose residuals are uncorrelated | shown valid in that case only; verify residual independence |
| hv-block CV (Racine 2000) | dependent data, model selection | remove a gap of h observations on each side of the test block from training |
| Purged / embargoed k-fold, combinatorial purged CV (López de Prado 2018) | labels spanning time windows overlap | purge training rows whose label windows overlap the test labels; embargo a buffer after each test block |
| Spatial block CV / buffered leave-one-out (Roberts et al. 2017) | spatial autocorrelation | block size or buffer at least the residual autocorrelation range; watch for forced extrapolation |
| Nearest-neighbour distance matching, NNDM / kNNDM (Milà et al. 2022; Linnenbrink et al. 2024) | the prediction area differs from the sample area | choose folds so that the distribution of test → training nearest-neighbour distances matches that of prediction → training distances |
| Nested CV (Varma & Simon 2006) | hyperparameters or features are tuned on the same data | tune in the inner loop, estimate in the outer loop; otherwise the estimate is optimistically biased |

Combinations are common and often necessary: rolling origin *and* entity groups (forecast
existing and new entities), time blocks *and* spatial blocks, purging *within* rolling origin
when targets are multi-step.

## 5. Step 4 — check the chosen split against the data

Data-driven acceptance checks for a candidate split:

1. **Distance matching.** The distribution of test → training distances (in time, space or
   feature space) resembles the prediction → training distribution from step 1. This is the
   NNDM idea, generalised beyond geography: a rolling origin with the production retrain
   cadence reproduces the time distance automatically; group or spatial blocking reproduces
   entity or place novelty.
2. **Composition.** The share of new / short-history entities, regimes (seasons, weekdays),
   and key segments in the test folds resembles deployment.
3. **Independence.** Residual correlation across the split boundary is negligible (or a gap /
   buffer / purge removes it).
4. **Enough independent units.** Count the independent blocks (weeks, spatial blocks, groups)
   per partition; with fewer than ~20, intervals become wide — say so before running.
5. **Stability.** Re-run the baseline on the candidate split and on a stricter one; if the
   baseline's error jumps, the looser split leaks.

Report which candidates were considered, the evidence, and the choice.

## 6. Step 5 — derive selection and verification partitions

Split the evaluation folds into two **disjoint** partitions — **selection** (to choose) and
**verification** (to confirm) — each representative of deployment:

- forecasting: alternate time blocks (e.g. odd vs. even months) so both contain every season,
  rather than first vs. second half;
- groups: assign whole groups at random, stratified by size or segment;
- space: assign whole spatial blocks, interleaved across the area.

Resample the same independent unit for the significance test (weeks, spatial blocks, groups;
see `evaluation-protocol.md`). If hyperparameters are tuned, tune within the selection
partition only (nested), never on verification. Optionally keep a final untouched test set for
the very end of a project.

## 7. Worked example (next-day demand per entity)

The data: daily demand for a few hundred geographically clustered entities, about one year of
training history before the evaluation year, demand strongly tied to temperature and the
seasonal profile, and new entities added in waves.

- *Deployment*: forecast the next day's demand per entity once a day; target data up to T−2;
  new entities open in waves, some in areas without existing entities; one exogenous shock
  (weather) hits all entities on the same day.
- *Dependence*: strong shared daily shocks (the exogenous drivers explained ~70 % of the
  day-to-day variation of the total), persistence over several days; entities cluster in
  space.
- *Choice*: rolling origin over a full year (every operating day a target), retrain cadence as
  in production, segments by history at forecast time, significance by resampling calendar
  weeks, selection = odd months, verification = even months.
- *What it caught*: entities added in an area without training coverage appeared as a large
  error in one month — rolling origin reproduced the spatial novelty because it happened in
  time. A random entity-day k-fold would have hidden it (neighbouring entities and days in
  training), and would have rated location features far too highly (entity-level R² 0.69 for
  latitude and longitude with random folds, while location over-predicted the new area's
  level by 155 %).

## 8. References

- Roberts, D. R. et al. (2017). Cross-validation strategies for data with temporal, spatial,
  hierarchical, or phylogenetic structure. *Ecography* 40(8), 913–929.
- Tashman, L. J. (2000). Out-of-sample tests of forecasting accuracy: an analysis and review.
  *International Journal of Forecasting* 16(4), 437–450.
- Bergmeir, C., Hyndman, R. J., Koo, B. (2018). A note on the validity of cross-validation
  for evaluating autoregressive time series prediction. *Computational Statistics & Data
  Analysis* 120, 70–83.
- Racine, J. (2000). Consistent cross-validatory model-selection for dependent data: hv-block
  cross-validation. *Journal of Econometrics* 99(1), 39–61.
- López de Prado, M. (2018). *Advances in Financial Machine Learning*, Wiley — ch. 7 (purged
  k-fold, embargo) and ch. 12 (combinatorial purged CV).
- Milà, C. et al. (2022). Nearest neighbour distance matching Leave-One-Out Cross-Validation
  for map validation. *Methods in Ecology and Evolution*.
- Linnenbrink, J., Milà, C., Ludwig, M., Meyer, H. (2024). kNNDM CV: k-fold nearest-neighbour
  distance matching cross-validation for map accuracy estimation. *Geoscientific Model
  Development* 17, 5897–5912.
- Varma, S., Simon, R. (2006). Bias in error estimation when using cross-validation for model
  selection. *BMC Bioinformatics* 7, 91.
