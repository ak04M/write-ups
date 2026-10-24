# Heavy Hand

## Challenge Description

Aurora Health's pharmacy records may reveal a prescriber dispensing unusually large quantities of controlled substances. The objective was to identify the prescriber with the highest total quantity of DEA Schedule II medications prescribed during the fourth quarter of 2025 and retrieve their DEA number.

## Solution

First, I switched to the `aurora_pharmacy` database and inspected the relevant tables to identify the drug schedules, prescription quantities, prescriber IDs, DEA numbers, and prescription dates.

I joined `prescriptions` with `drugs` and `prescribers`, filtered for Schedule II drugs prescribed between October 1 and December 31, 2025, and summed the quantities for each prescriber.

```sql
USE aurora_pharmacy;

SELECT
    p.dea_number,
    SUM(rx.quantity) AS total_quantity
FROM prescriptions rx
JOIN drugs d
    ON rx.drug_id = d.drug_id
JOIN prescribers p
    ON rx.prescriber_id = p.prescriber_id
WHERE d.dea_schedule = 'II'
  AND rx.written_date >= '2025-10-01'
  AND rx.written_date < '2026-01-01'
GROUP BY p.prescriber_id, p.dea_number
ORDER BY total_quantity DESC
LIMIT 1;
```

The results were sorted by total quantity in descending order, and `LIMIT 1` returned the prescriber with the highest total.

**Flag format:** `deadface{BR2938471}`
