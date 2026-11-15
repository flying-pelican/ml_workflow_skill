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
- **Open round:** challenger 7, the promotion lift scaled by store age at forecast time.
  Hypothesis: scaling the lift by store age removes the over-forecast of new stores in
  promotion weeks. Target slice: stores younger than 6 months in promotion weeks. Planned:
  one screening run on the new-store segment, then the full protocol; 7 € in total.
