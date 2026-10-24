# Ghost in Registry

## Challenge Description
Aurora's investigators suspected a fabricated patient who received Schedule II controlled-substance prescriptions but had never attended a clinical encounter. The goal was to identify the patient by finding prescription records without any matching encounter in `aurora_ehr`.

## Solution

I switched to the `aurora_pharmacy` database and joined prescriptions with the drug and patient tables. Since Aurora uses a shared `patient_id` across systems, I could check the patient's encounter history in `aurora_ehr`.

```sql
USE aurora_pharmacy;

SELECT
    p.patient_id,
    p.mrn,
    SUM(rx.quantity) AS total_quantity
FROM prescriptions rx
JOIN drugs d
    ON rx.drug_id = d.drug_id
JOIN aurora_ehr.patients p
    ON rx.patient_id = p.patient_id
WHERE d.dea_schedule = 'II'
  AND NOT EXISTS (
      SELECT 1
      FROM aurora_ehr.encounters e
      WHERE e.patient_id = p.patient_id
  )
GROUP BY p.patient_id, p.mrn
ORDER BY total_quantity DESC;
```

The query returned one patient with Schedule II prescriptions totaling 540 units but no recorded clinical encounters.

## Flag

```text
deadface{AH-0090001}
```
