# Too Many Places at Once

## Challenge Description
Aurora Health's incident-response team suspected a compromised account after a DEADFACE post described one legitimate user logging in from two different workstations at the same timestamp, repeatedly. The goal was to identify the account responsible.

## Solution

First, I selected the `aurora_audit` database and inspected the available tables and the structure of `access_log` to identify the relevant fields, including `user_id`, `workstation`, and `accessed_at`.

```sql
USE aurora_audit;

DESCRIBE access_log;
```

Next, I joined `access_log` with `users` and grouped the records by username and timestamp. Using `COUNT(DISTINCT workstation)`, I filtered for timestamps associated with at least two different workstations.

```sql
SELECT
    u.username,
    a.accessed_at,
    COUNT(DISTINCT a.workstation) AS workstation_count,
    GROUP_CONCAT(DISTINCT a.workstation) AS workstations
FROM access_log a
JOIN users u
    ON a.user_id = u.user_id
GROUP BY u.user_id, u.username, a.accessed_at
HAVING COUNT(DISTINCT a.workstation) >= 2
ORDER BY workstation_count DESC, a.accessed_at;
```

The results showed that `kbhatt` repeatedly accessed the system from two different workstations at the exact same timestamp across multiple dates, identifying the compromised account.

## Flag

```text
deadface{kbhatt}
```
