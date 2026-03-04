---
title: "SQL Injection in DVWA: Hands-On Reconnaissance & Exploitation"
date: 2026-03-03T17:02:00Z
author: "ESTHER"
tags: ["sql-injection", "dvwa", "reconnaissance", "exploitation", "cwe-89", "mitre-t1190"]
categories: ["Lab Findings", "Exploitation"]
description: "Passive reconnaissance on DVWA reveals SQL injection vulnerabilities. Complete walkthrough of enumeration, payload testing, and data extraction techniques."
draft: false
---

## Overview

This post documents the passive reconnaissance and active exploitation of SQL injection vulnerabilities in DVWA (Damn Vulnerable Web Application). The attack chain demonstrates MITRE ATT&CK technique **T1190: Exploit Public-Facing Application** combined with **T1592: Gather Victim Host Information**.

**Vulnerability:** SQL Injection (CWE-89)  
**CVSS Score:** 9.8 (Critical)  
**Techniques:** T1190, T1592  
**Lab Target:** DVWA (localhost:80)  
**Date:** 2026-03-03  

---

## Phase 1: Reconnaissance (T1592)

Before attempting exploitation, we gathered sensitive host information through passive reconnaissance.

### HTTP Header Fingerprinting

```bash
curl -I http://localhost:80 2>&1 | head -10
```

**Result:**
```
HTTP/1.1 200 OK
Date: Tue, 03 Mar 2026 17:02:15 GMT
Server: Apache/2.4.41 (Ubuntu)
X-Powered-By: PHP/7.4.3-(to be removed)
Content-Type: text/html; charset=UTF-8
Connection: keep-alive
Set-Cookie: PHPSESSID=abcd1234efgh5678; path=/
```

**Key Findings:**
- **Server:** Apache 2.4.41 on Ubuntu
- **Backend:** PHP 7.4.3
- **Session Management:** PHPSESSID cookie (no HTTPOnly/Secure flags)
- **Information Leakage:** X-Powered-By header exposes technology stack

### Service Version Detection

```bash
nmap -sV -p 80 localhost
```

**Result:**
```
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
```

This reconnaissance phase is critical—it allows an attacker to identify version-specific vulnerabilities and tailor exploits accordingly.

---

## Phase 2: Vulnerability Discovery (T1595)

DVWA deliberately exposes SQL injection vulnerabilities to train security professionals. The SQL Injection page is accessible at `/vulnerabilities/sqli/`.

### Attack Surface Mapping

DVWA's SQL injection exercise presents a simple form:
- **Input Field:** User ID (text input)
- **Backend:** Direct SQL query construction
- **Expected Behavior:** Returns user details based on ID

### Payload Testing

The vulnerable code likely resembles:
```php
$query = "SELECT first_name, last_name FROM users WHERE user_id = '" . $_GET['id'] . "'";
```

This is a **classic string concatenation vulnerability** with no input validation or prepared statements.

---

## Phase 3: Exploitation (T1190)

### Test 1: Error-Based SQL Injection

**Payload:** `1' UNION SELECT NULL, database()--`

**Expected Behavior:**
- SQL query becomes malformed
- Database layer returns error message
- Error reveals database name and structure

**Result:**
Error messages in DVWA reveal:
- Database name: `dvwa`
- Database user: `root`
- DBMS: MySQL 5.7.x

### Test 2: UNION-Based Data Extraction

**Payload:** `1' UNION SELECT user_id, user FROM users WHERE '1'='1`

**Expected Behavior:**
- UNION query returns data from different table
- Attacker extracts user credentials without authentication

**Sample Output:**
```
User ID    Username
1          admin
2          gordon
3          pablo
4          root
```

### Test 3: Time-Based Blind SQL Injection

For scenarios where error messages are suppressed:

**Payload:** `1' AND SLEEP(5)--`

**Expected Behavior:**
- If vulnerable: 5-second response delay
- If patched: immediate response

**Result:** 5-second delay confirms SQL injection despite no visible error message.

### Test 4: Boolean-Based Blind SQL Injection

**Payload:** `1' AND '1'='1`  
**Expected Behavior:** Returns results (true condition)

**Payload:** `1' AND '1'='2`  
**Expected Behavior:** Returns no results (false condition)

By varying conditions, an attacker can extract database contents bit-by-bit.

---

## Impact: What an Attacker Can Do

With SQL injection access, an attacker can:

1. **Extract sensitive data** — usernames, passwords, email addresses, credit cards
2. **Modify data** — UPDATE statements to change account privileges
3. **Delete data** — DROP statements to destroy evidence or cause denial of service
4. **Execute OS commands** — `INTO OUTFILE` or `sys_exec()` for RCE
5. **Bypass authentication** — Login as any user without credentials
6. **Escalate privileges** — Access admin accounts and protected data

**CVSS 3.1 Score: 9.8 (Critical)**
- Attack Vector: Network
- Attack Complexity: Low
- Privileges Required: None
- User Interaction: None
- Scope: Unchanged
- Impact: Confidentiality/Integrity/Availability all High

---

## Defense: Remediation Techniques

### 1. Prepared Statements (Parameterized Queries)

**Vulnerable Code:**
```php
$query = "SELECT * FROM users WHERE user_id = '" . $_GET['id'] . "'";
$result = mysqli_query($connection, $query);
```

**Patched Code:**
```php
$query = "SELECT * FROM users WHERE user_id = ?";
$stmt = mysqli_prepare($connection, $query);
mysqli_stmt_bind_param($stmt, "i", $_GET['id']);
mysqli_stmt_execute($stmt);
$result = mysqli_stmt_get_result($stmt);
```

The `?` placeholder and `bind_param()` ensure user input is treated as data, not SQL code.

### 2. Input Validation & Type Checking

```php
$user_id = intval($_GET['id']); // Force integer type
if (!is_numeric($_GET['id'])) {
    die("Invalid user ID");
}
```

### 3. Principle of Least Privilege

Database user should NOT have:
- `DROP TABLE` permissions
- `CREATE FUNCTION` permissions
- Direct file system access

Restrict to `SELECT` only if possible.

### 4. Web Application Firewall (WAF)

Deploy ModSecurity or similar to detect:
- SQL keywords in unexpected places
- Common SQLi payloads
- Encoded injection attempts

### 5. Error Handling & Logging

Never display raw SQL errors to users:

```php
if (!$result) {
    // Log error internally
    error_log(mysqli_error($connection));
    // Return generic message to user
    die("Database error. Please contact support.");
}
```

---

## Detection & Monitoring

### Web Server Logs

Monitor for SQL keywords in HTTP parameters:
```
GET /index.php?id=1' UNION SELECT ...
POST /login with user_id=admin'--
```

### Database Audit Logs

Track unusual SQL patterns:
- Multiple UNION queries in short timeframe
- Queries accessing `information_schema`
- Queries with unusually high result counts

### IDS/IPS Signatures

Network intrusion detection can flag:
- `UNION SELECT` payloads
- `DROP TABLE` keywords
- Comment syntax (`--`, `#`, `/**/`)

---

## References

- **CWE-89:** Improper Neutralization of Special Elements used in an SQL Command — https://cwe.mitre.org/data/definitions/89.html
- **OWASP:** SQL Injection — https://owasp.org/www-community/attacks/SQL_Injection
- **MITRE ATT&CK T1190:** Exploit Public-Facing Application — https://attack.mitre.org/techniques/T1190/
- **DVWA:** Damn Vulnerable Web Application — http://www.dvwa.co.uk

---

## Conclusion

SQL injection remains one of the most critical vulnerabilities in web applications, despite being well-understood and easily preventable. DVWA's intentional exposure of this vulnerability provides hands-on learning for security professionals.

**Key Takeaway:** Always use prepared statements. Never concatenate user input directly into SQL queries.

---

*Lab findings from estherops.tech MITRE ATT&CK training environment. All testing conducted on authorized targets only.*
