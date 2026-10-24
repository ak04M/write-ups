# DEADFACE CTF — Prying Eyes

## Solution

The objective was to identify the user account that accessed the VIP patient's chart most frequently.

### 1. Identify the VIP's MRN

Using the result from **Admitted Under Another Name**, we obtained the VIP patient's MRN:

`AH-0044719`

### 2. Query the audit log

We switched to the `aurora_audit` database and identified the `access_log` and `users` tables. After inspecting their schemas, we joined them using `user_id` and counted accesses for the VIP's MRN.

```sql
SELECT u.username, u.full_name, COUNT(*) AS access_count
FROM access_log a
JOIN users u ON a.user_id = u.user_id
WHERE a.patient_mrn = 'AH-0044719'
GROUP BY u.user_id, u.username, u.full_name
ORDER BY access_count DESC;
```

### 3. Identify the account

The results were sorted by access count in descending order. The username in the first row was the account that accessed the VIP patient's chart most frequently.

**Flag:** `deadface{rborden}`

Replace `username` with the actual username from the query results.
