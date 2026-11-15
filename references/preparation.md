# Preparation: steps 0–6

Done once per project, and revisited when the use case, the data or the deployment changes.
`SKILL.md` holds the binding rules; this file holds the choices behind them that are easy to
get wrong. Record every agreement in the project's notes.

## Step 0: use case

- Ask for the metric definitions already in use (reports, dashboards, today's method), the
  level at which the primary metric is computed and how entities are weighted, and how the
  business reads the result (e.g. "units off per day"), so results can also be reported that
  way.
- Write acceptance checks as testable bands, e.g. "90 % interval coverage within 85–95 %",
  "bias within ±5 %", "price response monotonic and smooth on a price grid for every
  product". A challenger that fails one is not compared on accuracy at all.
- Say when a requirement conflicts with another choice (a strictly monotonic response with a
  model class that cannot be constrained).
- If today's method is documented ambiguously, ask rather than guess: a baseline weaker than
  the real method inflates every later gain.

## Step 1: data, stack and infrastructure

- Find out who executes: you directly, or the user runs the cells you write. A user-executed
  run adds a round trip to every cycle.
- Find out whether compute is already paid (local hardware, a reservation: an extra run costs
  wall-clock time) or pay-per-use (every run and every parallel candidate costs money).
- The valuation question, with example answers to offer: "an hour of my time is worth about
  100 € of compute", "compute is idle, my time is the bottleneck", "the cloud budget is capped
  at 200 € a month". Example working rule: "up to three candidates in parallel locally;
  tuning only after the feature set is stable".
- Size the evaluation to the run budget, and say what the budget costs in statistical power.

## Cycle length

The time from deciding a change to its result in front of the user: data preparation,
training of all refits, scoring, bootstrap, review. A slow loop quietly pushes towards
bundling several changes into one iteration, which makes results uninterpretable.

- Too long: screening runs on a cheaper configuration (coarser refits, a stratified subsample,
  fewer bootstrap draws, cached features), the full protocol only for candidates that might be
  promoted. Say what each shortcut costs (a subsample widens the intervals; a coarser cadence
  hides effects of new entities).
- Very short and cheap: the risk turns around. Many quick challengers against the same
  selection partition inflate false wins. Keep verification untouched by tuning, keep the
  history complete including rejected ideas, and consider a final holdout for the end of the
  project.

## Step 2: explore the data

Each finding must end up as the reason for a design choice. The checks that are easy to miss:

- **System-wide anomalies** (outages, delivery failures): flag periods whose total falls far
  below a robust local reference, e.g. below 10 % of the centred 15-day median. Exclude them
  from training and evaluation. Check the near-misses by hand: a partial shutdown just above
  the threshold can still teach a wrong effect.
- **Pseudo-entities** that are not real demand (test, depot, internal accounts): remove them
  everywhere. Decide whether zero means no demand or not operating.
- **History length at forecast time** becomes the segment definition. Entities older than the
  data have an unknown age; counting age from the data start turns it into a time index.
- **Trend** on a fixed set of entities that existed throughout; otherwise additions
  masquerade as trend.
- **Drivers:** explained variation of the total by calendar alone, exogenous drivers alone and
  both. If the season is mostly a proxy for an exogenous driver, model the driver: with one
  year of history a calendar season feature replays that single year. Compare each holiday
  separately with the same weekday a week before and after; effects differ widely.
- **Availability at decision time:** for every exogenous driver, archived forecasts against
  observations, and the share of the observed effect the forecast retains. Observations give
  an upper bound only.
- **Variance decomposition** of the target into entity, shared date and residual effects: a
  large entity share means level and location features matter.
- **New entities:** ramp-up against established entities over the same calendar weeks, so
  season does not masquerade as ramp-up; and the hard case: trained on established entities,
  does it predict a new area?

**Hand-over** to the later steps:

- the *potential and challenges* summary: what drivers explain beyond simple history, which
  segments simple rules will serve badly, the rough achievable gain; cold starts, anomalies,
  short history, drivers available only as forecasts, dependence, data that cannot leave its
  environment;
- the exclusion rules and the cleaned data;
- the segment definition and the entity lifecycle facts;
- the dominant drivers and whether each is available at decision time;
- the dependence measurements for the split design (`references/evaluation.md`);
- candidate features for iteration 0, each with the finding behind it, and the full list of
  findings a model could address, as input for the exploratory shot.

## Step 4: non-ML baseline

- Check the reproduction of today's method against its historical outputs where they exist.
- Make the fallbacks of every benchmark for entities with little or no history explicit and
  document them.
- Name each benchmark's weaknesses (lags the season, blind to exogenous drivers, fails on new
  entities): they are where a model can add value. Where a benchmark is already hard to beat,
  say so early.

## Step 6: the exploratory shot

Right after iteration 0, one quick model that throws everything at the problem: all candidate
features, fallbacks for cold starts, forecasts of the exogenous drivers, flags for anomalies
or regime changes, a flexible model class with default settings. Written in one sitting,
time-boxed to a few cycle lengths.

It is not an iteration. Its gain cannot be attributed to single changes (interactions,
correlated features splitting importance), which is exactly why it cannot replace the loop.
Two rules hold, because without them its insights are wrong:

- **Same rows and information cutoff** as every evaluated model. A leak produces a false
  potential and sends the iterations after a ghost; if the shot is suspiciously good, check
  for leakage first.
- **Selection partition only.** Report verification at most as one overall number.

Read from it, against iteration 0 and the baseline by segment, month and slice:

- the size of the potential (if the shot barely beats iteration 0, the gain the use case needs
  may not be reachable with this data: say so early);
- which findings carry signal (features it relies on, where it gains most);
- what still fails, even for the shot (needs a diagnosis or new data, not more iterations);
- its acceptance checks (reported, not gating; a failure shows early that constraints are
  needed);
- its run time, as a first estimate of a full-featured model's cycle.

The output is the ranked candidate list: hypothesis, target slice, expected gain, expected
cost, ordered by expected information gain per run time. Record the shot once, in its own
section, and do not update it later.
