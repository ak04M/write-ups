# DEADFACE CTF — Identifying the Attending Physician

## 1. Database Enumeration

Connected to the provided MySQL shell and selected the EHR database:

```sql
USE aurora_ehr;
```

Identified the `encounters` table and inspected its schema:

```sql
DESCRIBE encounters;
```

The table contained `patient_id`, `attending_provider_id`, `encounter_type`, and `admit_date`, which were relevant to the investigation.

## 2. Find the VIP's Admission

The VIP patient had `patient_id = 1007`. Queried their admissions during November 2025:

```sql
SELECT encounter_id, patient_id, attending_provider_id,
       encounter_type, admit_date, discharge_date, status
FROM encounters
WHERE patient_id = 1007
  AND admit_date >= '2025-11-01'
  AND admit_date < '2025-12-01';
```

This identified the admission and its attending provider ID.

## 3. Identify the Provider

Inspected the `providers` table and used the attending provider ID to retrieve the corresponding provider record and system username.

## 4. Flag

Submitted the identified username in the required format:

`deadface{mreyes}`
