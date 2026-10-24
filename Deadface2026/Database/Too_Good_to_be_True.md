# Too Good to be True

## Challenge Description
The Veltrexa clinical trial contained a suspected fabricated efficacy result. The objective was to identify the subject whose `primary_efficacy_delta` exceeded the study-wide mean by more than three standard deviations.

## Solution

I switched to the `aurora_trials` database and calculated the mean and population standard deviation of all primary efficacy results. I then joined `results` with `subjects` to identify any subject whose recorded value exceeded the three-standard-deviation threshold.

```sql
USE aurora_trials;

SELECT
    s.subject_code,
    r.value AS efficacy_delta,
    stats.mean_value,
    stats.stddev_value
FROM results r
JOIN subjects s
    ON r.subject_id = s.subject_id
CROSS JOIN (
    SELECT
        AVG(value) AS mean_value,
        STDDEV_POP(value) AS stddev_value
    FROM results
    WHERE measure = 'primary_efficacy_delta'
) stats
WHERE r.measure = 'primary_efficacy_delta'
  AND r.value > stats.mean_value + 3 * stats.stddev_value
ORDER BY r.value DESC;
```

The query identifies the subject whose efficacy improvement lies more than three standard deviations above the mean. The returned `subject_code` is the required flag value.

## Flag

```text
deadface{VELT-0148}
```

Replace `SUBJECT_CODE` with the value returned by the query.
