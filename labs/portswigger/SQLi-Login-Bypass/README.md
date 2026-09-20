# PortSwigger Web Security Academy - SQL Injection Vulnerability Allowing Login Bypass

**Category:** Web / SQL Injection
**Lab Difficulty:** Apprentice
**Lab Link:** https://portswigger.net/web-security/sql-injection/lab-login-bypass

## Lab Scenario

> This lab contains a SQL injection vulnerability in the login function. To solve the lab, perform a SQL injection attack that logs in to the application as the `administrator` user.

## Recon

The login form submits `username` and `password` as `application/x-www-form-urlencoded` POST parameters, along with a CSRF token bound to the current session:

## The Vulnerability

**SQL Injection in the login query (CWE-89).**

The `username` parameter is concatenated directly into the backend SQL query without sanitization or parameterization. A single quote in the username field breaks out of the string literal.

### Identifying the injection

Submitting a lone quote as the username:

returned an **HTTP 500 Internal Server Error** — confirming the input reaches the SQL query unsanitized and broke its syntax.

### Building the bypass payload

The initial attempt assumed the query wrapped the username in parentheses:

This also returned a 500 — indicating the assumption was wrong (an unmatched closing parenthesis, with no corresponding opening one in the real query, was itself causing the syntax error).

Simplifying and retesting without the parenthesis assumption:

This resolved the query cleanly. The resulting query is logically equivalent to:

```sql
SELECT * FROM users WHERE username = 'admin' or '1'='1'-- -' AND password = 'anything'
```

`'1'='1'` is always true, so the `OR` condition makes the `WHERE` clause match a row regardless of the real credentials, and `-- -` comments out the rest of the original query (the password check) entirely.

## Exploitation

Submitting the payload above returned:

A new authenticated session was issued for the `administrator` account, without ever supplying valid credentials. Following the redirect confirmed:

Note: this bypass had to be performed through the actual browser (not a separate `curl`/Burp session) for the lab's solve-tracking to register against the correct session — the injection itself succeeds identically either way, but the lab only marks itself "Solved" for the session that receives the resulting cookie.

## Result

**Lab solved** — authenticated as `administrator` via SQL injection login bypass, no valid password required.

## Root Cause & Fix

- **Never build SQL queries via string concatenation of user input.** Use parameterized queries / prepared statements, which treat user input strictly as data and never as part of the query's syntax — this eliminates the entire class of bug regardless of what characters an attacker supplies.
- If an ORM or query builder is in use, avoid any "raw query" escape hatches for user-controlled values.
- Defense in depth: enforce least-privilege database accounts for the application (a compromised login query shouldn't be able to read/write arbitrary tables), and log/alert on SQL syntax errors returned to the application layer, since a 500 caused by malformed injected SQL is a strong signal of an active injection attempt.

## References

- [PortSwigger: SQL injection vulnerability allowing login bypass](https://portswigger.net/web-security/sql-injection/lab-login-bypass)
- [PortSwigger: SQL injection cheat sheet](https://portswigger.net/web-security/sql-injection/cheat-sheet)
- [OWASP: SQL Injection Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html)
