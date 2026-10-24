# DEADFACE CTF — Pharmacy Formulary

## Objective
Count the number of drugs classified as DEA Schedule II in Aurora Health's pharmacy formulary.

## Enumeration

First, switch to the pharmacy database and inspect the `drugs` table:

```sql
USE aurora_pharmacy;
DESCRIBE drugs;
```

After identifying the column containing the DEA schedule, count the Schedule II drugs:

```sql
SELECT COUNT(*)
FROM drugs
WHERE dea_schedule = 'II';
```

## Result

The query returns the total number of Schedule II drugs in the formulary.

**Flag:** `deadface{9}`

Replace `COUNT` with the number returned by the query.
