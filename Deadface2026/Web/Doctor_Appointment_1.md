# DEADFACE CTF — Health Spark

## Challenge Overview
- **Target:** `http://45.79.200.170:3000/`
- **Application:** Health Spark — Doctor Appointment
- **Objective:** Find the flag by exploiting a weakness in the application's login flow.
- **Flag format:** `deadface{flag-text}`

## Solution

### 1. Inspect the Login Page
Opened the Health Spark application in the browser. A login page appeared, requesting a username and password.

### 2. Try Default Credentials
Attempted to log in using the common default credentials:

```text
Username: admin
Password: admin
```

The login attempt failed.

### 3. Test for SQL Injection
Next, tried a SQL injection payload in the login form. The attempt did not immediately provide access, but the application returned a **different error message**.

This error provided a clue that the HTTP request itself needed further inspection.

### 4. Intercept the Request Using Burp Suite
Opened Burp Suite and intercepted the login request. Sent the request again with the same SQL injection payload.

### 5. Retrieve the Flag
Inspected the HTTP response in Burp Suite. The response contained the challenge flag.

```text
deadface{flag-text}
```

*Replace the placeholder above with the exact flag obtained from the response.*

## Attack Path

Login page → Default credentials failed → SQL injection attempt → Different error message → Intercept request in Burp Suite → Resend request with the same payload → Flag revealed in the HTTP response

## Key Takeaway
An application's error messages can reveal useful clues during testing. Inspecting the raw HTTP request and response in Burp Suite helped identify the behavior that exposed the flag.
