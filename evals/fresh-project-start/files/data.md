# Customer table export (exported 31 October 2026)

48,213 customers of the subscription service, one row per customer. In the last 12 months,
6.1 % of them cancelled.

| Column | Description |
|---|---|
| customer_id | unique id |
| signup_date | date of the first subscription |
| plan | basic, standard or premium |
| monthly_fee | current monthly fee in € |
| region | one of 9 sales regions |
| logins_last_30d | number of logins in the 30 days before the export |
| support_tickets_last_90d | number of support tickets in the 90 days before the export |
| discount_active | whether a retention discount is currently active |
| cancellation_reason | reason given at cancellation, empty for active customers |
| churned | 1 if the customer cancelled in the 12 months before the export, else 0 |
