# DEADFACE CTF — Aurora Health Patient Registry

## Objective
Identify the high-profile patient who checked in under a fake name and retrieve their real Medical Record Number (MRN).

## 1. Database Enumeration

We were provided with a database shell. First, we checked the server version:

```sql
SELECT @@VERSION;
```

The output identified MySQL 8.0.46. Our initial attempts to use SQL Server-specific syntax failed, and `SELECT DATABASE()` returned `NULL`, indicating that no database was selected.

We listed the available databases:

```sql
SHOW DATABASES;
```

Among the results was `aurora_ehr`, which was relevant to the patient registry.

## 2. Locate the Patients Table

We selected the database and listed its tables:

```sql
USE aurora_ehr;
SHOW TABLES;
```

The database contained a `patients` table. We inspected its structure:

```sql
DESCRIBE patients;
```

The columns included `mrn`, `first_name`, `last_name`, `alias`, and `is_vip`.

## 3. Identify the VIP Patient

Instead of examining every patient manually, we filtered the records using the VIP flag:

```sql
SELECT patient_id, mrn, first_name, last_name, alias, is_vip
FROM patients
WHERE is_vip = 1;
```

The query returned one matching record:

| Field | Value |
|---|---|
| Patient ID | 1007 |
| Registered name | Gregory Halloran |
| Alias | George Harmon |
| VIP flag | 1 |
| MRN | AH-0044719 |

The `alias` field revealed the fake identity, while the `mrn` field provided the requested identifier.

## 4. Flag / Answer

**Real MRN: `AH-0044719`**

## Key Takeaway

The challenge was solved through basic database enumeration: identify the database, inspect the table schema, and filter records using the `is_vip` column. Understanding the database engine and its syntax was essential to resolving the initial errors.
