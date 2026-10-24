# Upon the Throne

## Challenge Description
Aurora's breach involved two insiders: an audit clerk who accessed a gubernatorial candidate's chart and a physician coerced into falsifying the Veltrexa clinical trial. The objective was to identify the common DEADFACE handler by comparing the `handler_ref` fields in the audit-system `users` table and the EHR `providers` table.

## Solution

I joined the two tables on `handler_ref` to find the codename present in both systems, excluding null and empty values.

```sql
SELECT DISTINCT u.handler_ref
FROM aurora_audit.users u
JOIN aurora_ehr.providers p
    ON u.handler_ref = p.handler_ref
WHERE u.handler_ref IS NOT NULL
  AND u.handler_ref <> '';
```

The query returned a single matching handler codename: `REVENANT-7`.

## Flag

```text
deadface{REVENANT-7}
```
