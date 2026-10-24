# After the Lock

## Challenge Description
Aurora Health's Veltrexa clinical trial database was locked on December 1, 2025. The objective was to identify the subject whose `primary_efficacy_delta` was modified after the official data-lock date.

## Solution

First, I inspected the `studies` table to find the Veltrexa study and its official lock date.

```sql
DESCRIBE studies;
SELECT * FROM studies;
```

The study had `study_id = 1` and a `data_lock_date` of `2025-12-01`.

Next, I queried the audit trail for changes to the primary efficacy endpoint:

```sql
SELECT
    dc.record_ref,
    dc.field_name,
    dc.old_value,
    dc.new_value,
    dc.changed_at,
    dc.reason,
    dc.study_id
FROM data_changes dc
WHERE dc.field_name = 'primary_efficacy_delta'
ORDER BY dc.changed_at;
```

The results revealed that subject `VELT-0137` had their primary efficacy result changed from `2.3` to `6.0` on December 6, 2025, after the database was locked. The change was recorded as *“Query resolution per monitor”*, making it the suspicious post-lock modification.

## Flag

```text
deadface{VELT-0137}
```
