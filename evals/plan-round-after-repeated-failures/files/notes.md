# Project notes: weekly demand forecast

## Agreements
- **Use case:** weekly demand per store and product family for a grocery chain, 4 weeks
  ahead, feeding replenishment.
- **Primary metric:** WAPE on store × family × week, weighted by volume.
- **Acceptance check:** coverage of the 80 % interval within 75–85 %.
- **Guardrail:** bias within ±3 % in each partition.
- **Split:** rolling origin over 2025 with weekly refits; selection = odd months,
  verification = even months; block bootstrap over calendar weeks.
- **Valuation:** "Cloud budget capped at 150 € per month; my time is worth more than compute,
  but stay within the cap." A full run costs about 40 minutes and 6 €, a run on one segment
  about 8 minutes and 1 €.

## State
- **Champion:** iteration 4, gradient boosting with price, promotion flags, a 4-week profile
  and weather forecasts.
- **History:**
  - Iteration 5 (promotion lift from the last 8 promotion weeks): rejected; stores opened in
    the last 6 months shown worse in verification.
  - Iteration 6 (promotion lift by product family): rejected for the same reason. Diagnosis:
    new stores get the promotion lift of established stores, but their customers react about
    half as strongly in their first months.
- **Budget:** 95 € of this month's budget are spent.
- **Open ideas:** (a) promotion lift scaled by store age at forecast time; (b) holiday
  features per holiday instead of one flag; (c) tuning the learning rate and tree depth;
  (d) a hierarchical reconciliation of store and region forecasts.
