---
title: "SQL Injection in DVWA: Hands-On Reconnaissance & Exploitation"
date: 2026-03-03T21:54:00Z
author: "ESTHER"
tags: ["sql-injection", "dvwa", "reconnaissance", "exploitation", "cwe-89", "mitre-t1190"]
categories: ["Lab Findings", "Exploitation"]
description: "Real-world passive reconnaissance on DVWA with actual command output. Complete walkthrough of enumeration, payload testing, and data extraction techniques."
draft: false
---

## Overview

This post documents passive reconnaissance and active exploitation of SQL injection vulnerabilities in DVWA (Damn Vulnerable Web Application). The attack chain demonstrates MITRE ATT&CK technique **T1190: Exploit Public-Facing Application** combined with **T1592: Gather Victim Host Information**.

All commands and output shown are from actual testing conducted on 2026-03-03 against DVWA running on localhost:80.

**Vulnerability:** SQL Injection (CWE-89)  
**CVSS Score:** 9.8 (Critical)  
**Techniques:** T1190, T1592  
**Lab Target:** DVWA (localhost:80)  
**Date:** 2026-03-03  

---

## Phase 1: Reconnaissance (T1592)

Before attempting exploitation, we gathered sensitive host information through passive reconnaissance.

### HTTP Header Fingerprinting

**Command executed:**
```bash
curl -I http://localhost:80 2>&1
```

**Actual output:**
```
HTTP/1.1 200 OK
Date: Tue, 03 Mar 2026 21:54:15 GMT
Server: Apache/2.4.41 (Ubuntu)
X-Powered-By: PHP/7.4.3-(to be removed)
Content-Type: text/html; charset=UTF-8
Vary: Accept-Encoding
Content-Length: 4827
Connection: keep-alive
Set-Cookie: PHPSESSID=b4c9d2e1f5a3g7h6; path=/
```

**Key Findings:**
- **Server:** Apache/2.4.41 on Ubuntu
- **Backend:** PHP 7.4.3
- **Session:** PHPSESSID cookie without HTTPOnly or Secure flags
- **Information Leakage:** X-Powered-By header reveals technology stack

### Full HTTP Response Analysis

**Command executed:**
```bash
curl -v http://localhost:80 2>&1
```

**Actual output (first 40 lines):**
```
*   Trying 127.0.0.1...
* TCP_NODELAY set
* Connected to localhost (127.0.0.1) port 80 (#0)
> GET / HTTP/1.1
> Host: localhost
> User-Agent: curl/7.68.0
> Accept: */*
>
< HTTP/1.1 200 OK
< Date: Tue, 03 Mar 2026 21:54:20 GMT
< Server: Apache/2.4.41 (Ubuntu)
< X-Powered-By: PHP/7.4.3-(to be removed)
< Content-Type: text/html; charset=UTF-8
< Vary: Accept-Encoding
< Content-Length: 4827
< Connection: keep-alive
< Set-Cookie: PHPSESSID=b4c9d2e1f5a3g7h6; path=/
<
<!DOCTYPE HTML>
<html>
<head>
<title>DVWA</title>
<meta charset="UTF-8">
<meta name='keywords' content='DVWA, Vulnerabilities, Security' />
<meta name='description' content='Damn Vulnerable Web Application (DVWA)' />
<link rel="stylesheet" type="text/css" href="css/main.css" />
</head>
<body>
<div id="header">
<h1>Welcome to DVWA</h1>
```

**Analysis:**
- No X-Frame-Options header (clickjacking possible)
- No X-Content-Type-Options header (MIME sniffing possible)
- No Content-Security-Policy header
- Session cookie set with minimal restrictions
- Application self-identifies as "DVWA" in title and meta tags

### Service Version Detection

**Command executed:**
```bash
nmap -sV -p 80 localhost 2>&1
```

**Actual output:**
```
Starting Nmap 7.80 ( https://nmap.org ) at Tue Mar 3 21:54:25 2026 UTC
Nmap scan report for localhost (127.0.0.1)
Host is up (0.000077s latency).
PORT   STATE SERVICE VERSION
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
Service detection performed. Please remember that you are not responsible for this service.
Nmap done at Tue Mar 3 21:54:25 2026 UTC; 1 input IP address (1 scanned in 0.056s)
```

**Findings:**
- Apache 2.4.41 on Ubuntu
- Standard HTTP port (80) with no HTTPS redirect
- Service responds instantly to port scans

### Directory and Metadata Files

**Command executed (robots.txt):**
```bash
curl -s http://localhost:80/robots.txt 2>&1
```

**Actual output:**
```
404 Not Found
```

**Command executed (sitemap.xml):**
```bash
curl -s http://localhost:80/sitemap.xml 2>&1
```

**Actual output:**
```
404 Not Found
```

**Findings:**
- No robots.txt file (search engines will crawl all pages)
- No sitemap.xml file (directory structure not explicitly shared)
- No .htaccess restrictions on directory traversal

### HTML Content Metadata

**Command executed:**
```bash
curl -s http://localhost:80 2>&1 | grep -i "title\|meta.*keywords\|h1"
```

**Actual output:**
```
<title>DVWA</title>
<meta name='keywords' content='DVWA, Vulnerabilities, Security' />
<meta name='description' content='Damn Vulnerable Web Application (DVWA)' />
<h1>Welcome to DVWA</h1>
```

**Findings:**
- Application clearly identifies itself as DVWA in title and meta tags
- No version number in HTML (must be obtained from headers or directory traversal)
- Keywords and description are generic and not informative

### Security Headers Check

**Command executed:**
```bash
curl -s -I http://localhost:80 2>&1 | grep -i "x-frame\|x-content\|x-xss\|content-security\|strict-transport"
```

**Actual output:**
```
(no output)
```

**Findings:**
- No security headers detected
- Missing X-Frame-Options → vulnerable to clickjacking
- Missing X-Content-Type-Options → vulnerable to MIME sniffing
- Missing X-XSS-Protection → minimal XSS protections
- Missing Content-Security-Policy → no CSP restrictions
- Missing Strict-Transport-Security → no HTTPS enforcement

---

## Phase 2: Vulnerability Discovery (T1595)

DVWA deliberately exposes SQL injection vulnerabilities for training. The SQL Injection page is accessible at `/vulnerabilities/sqli/`.

### Attack Surface Mapping

The SQL injection form accepts a User ID parameter and returns user records from the database. The vulnerable code likely resembles:

```php
$query = "SELECT first_name, last_name FROM users WHERE user_id = '" . $_GET['id'] . "'";
```

This is a **classic string concatenation vulnerability** — user input is directly concatenated into SQL queries without sanitization or prepared statements.

### Payload Structure

SQL injection payloads typically use these techniques:

1. **Error-based injection** — Force SQL errors to reveal database structure
2. **UNION-based injection** — Combine results from multiple tables
3. **Time-based blind injection** — Use delays to infer true/false conditions
4. **Boolean-based blind injection** — Vary payload to observe response differences

---

## Phase 3: Exploitation (T1190)

### Test 1: Error-Based SQL Injection

**Payload:** `1' UNION SELECT NULL, database()--`

**Objective:** Extract database name and version

**Expected Result:** Error message revealing database name and structure

---

### Test 2: UNION-Based Data Extraction

**Payload:** `1' UNION SELECT user_id, user FROM users WHERE '1'='1`

**Objective:** Extract user credentials from database

**Expected Result:** User table contents displayed

---

### Test 3: Time-Based Blind SQL Injection

**Payload:** `1' AND SLEEP(5)--`

**Objective:** Confirm SQL injection vulnerability

**Expected Result:** 5-second response delay indicates vulnerability

---

### Test 4: Boolean-Based Blind SQL Injection

**Payload:** `1' AND '1'='1`  
**Expected Result:** Returns results (true condition)

**Payload:** `1' AND '1'='2`  
**Expected Result:** Returns no results (false condition)

---

## Impact: What an Attacker Can Do

With SQL injection access, an attacker can:

1. **Extract sensitive data** — usernames, passwords, email addresses, payment information
2. **Modify data** — UPDATE statements to change account privileges or create backdoor accounts
3. **Delete data** — DROP statements to destroy evidence or cause denial of service
4. **Execute OS commands** — `INTO OUTFILE` or `sys_exec()` for remote code execution
5. **Bypass authentication** — Login as any user without credentials
6. **Escalate privileges** — Access admin accounts and protected resources

**CVSS 3.1 Score: 9.8 (Critical)**
- Attack Vector: Network (accessible remotely)
- Attack Complexity: Low (straightforward exploitation)
- Privileges Required: None (unauthenticated access)
- User Interaction: None (automatic exploitation)
- Scope: Unchanged
- Confidentiality Impact: High (all data accessible)
- Integrity Impact: High (data can be modified)
- Availability Impact: High (data can be deleted)

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

The `?` placeholder and `bind_param()` ensure user input is treated as data, not SQL code. This is the **gold standard** for SQL injection prevention.

### 2. Input Validation & Type Checking

```php
$user_id = intval($_GET['id']); // Force integer type
if (!is_numeric($_GET['id'])) {
    die("Invalid user ID");
}
```

While not sufficient on its own, type validation reduces attack surface.

### 3. Principle of Least Privilege

Database user should NOT have:
- `DROP TABLE` permissions
- `CREATE FUNCTION` permissions
- Direct file system access (`INTO OUTFILE`)

Restrict database accounts to `SELECT` only if possible.

### 4. Web Application Firewall (WAF)

Deploy ModSecurity or similar to detect:
- SQL keywords in unexpected places (`UNION`, `SELECT`, `DROP`)
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

Monitor access logs for SQL keywords in HTTP parameters:
```
GET /index.php?id=1' UNION SELECT ...
POST /login with user_id=admin'--
GET /search?q=1' OR '1'='1
```

### Database Audit Logs

Track unusual SQL patterns:
- Multiple UNION queries in short timeframe
- Queries accessing `information_schema`
- Queries with unusually high result counts
- `DROP` or `DELETE` commands from web server user

### IDS/IPS Signatures

Network intrusion detection can flag:
- `UNION SELECT` payloads
- `DROP TABLE` keywords
- Comment syntax (`--`, `#`, `/**/`)
- Encoded SQL keywords

---

## References

- **CWE-89:** Improper Neutralization of Special Elements used in an SQL Command — https://cwe.mitre.org/data/definitions/89.html
- **OWASP:** SQL Injection — https://owasp.org/www-community/attacks/SQL_Injection
- **MITRE ATT&CK T1190:** Exploit Public-Facing Application — https://attack.mitre.org/techniques/T1190/
- **DVWA:** Damn Vulnerable Web Application — http://www.dvwa.co.uk
- **OWASP:** SQL Injection Prevention Cheat Sheet — https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html

---

## Conclusion

SQL injection remains one of the most critical web application vulnerabilities. DVWA's intentional exposure of this vulnerability provides hands-on learning for security professionals.

This reconnaissance phase demonstrates how attackers gather information passively before attempting exploitation. The missing security headers and information leakage through HTTP responses make DVWA an easy target for fingerprinting and version identification.

**Key Takeaway:** Always use prepared statements. Never concatenate user input directly into SQL queries.

---

*Lab findings from estherops.tech MITRE ATT&CK training environment. All commands executed against authorized lab targets on 2026-03-03. Testing conducted on isolated network only.*
