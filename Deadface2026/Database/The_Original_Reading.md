# DEADFACE CTF — The Original Reading

## Approach

### 1. Inspect the trial database

Selected the `aurora_trials` database and listed its tables to locate `data_changes`, which stores the EDC audit trail.

```sql
USE aurora_trials;
SHOW TABLES;
DESCRIBE data_changes;
```

The schema revealed fields including `study_id`, `table_name`, `field_name`, `old_value`, `new_value`, and `reason`.

### 2. Investigate suspicious changes

An initial search for fields containing `endpoint` returned no results. This indicated that the manipulated endpoint was recorded under a different field name.

I then inspected the study records and queried the audit trail, examining the field names, old and new values, and change reasons to identify the suspicious efficacy-related edit associated with Veltrexa.

```sql
SELECT change_id, study_id, table_name, record_ref,
       field_name, old_value, new_value, changed_by,
       changed_at, reason
FROM data_changes
ORDER BY changed_at;
```

### 3. Recover the original value

The relevant audit entry records both the overwritten value (`2.3`) and the replacement (`6.0`). The challenge asks for the original value, so the required answer is the exact contents of `old_value` from the fraudulent edit.

**Flag format:** `deadface{2.3}`
