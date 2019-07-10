# Secure Coding with Python.

## Chapter 2: SQL Injection
### Requirement
Since we are creating a marketplace application, we first decide to allow the upload of Listings, just text. We will worry about users later, since we want to focus on getting the DB and Models setup without needed to worry about authentication and session management at this point.

### Development
Since the application will need some more configuration we change the `marketplace/__init__.py` to make use of the `create_app` factory function. We add the DB connection functions into `marketplace/db.py` and add the factory function. We also add the DB schema in `schema.sql` and add a flask command to init the DB, which we run with the `python -m flask init-db` command.

### Vulnerability
Since we are generating the SQL to insert the new listing in a very unsecure way, we can insert SQL commands that will be run in the DB. For example if we insert `'` as title or description we will get `psycopg2.errors.SyntaxError: INSERT has more target columns than expressions LINE 1: INSERT INTO listings (title, description) VALUES (''', ''') ^` instead of a success.

We can for example get the postgresql version or any other SQL function result, to check that out, insert `injection', (select version()))-- -` as the title. When we do so, the SQL that's going to be executed will be the following:
```sql
INSERT INTO listings (title, description) VALUES ('injection', (select version()))-- -', 'ignored description')
```
As it can be seen, the inserted title will be `injection` and the description will be the result of the `select version()` command, or any other command we wish to insert there, including dropping the DB.

### Testing
Testing for SQL injections is a tedious job, it's mostly done by hand or using special scanners, like web scanners or SAST/DAST tools. For this chapter we will be writing a very simple fuzzer function and create unit tests that use them in order to test for injections.

The fuzzer helper looks like this:
```python
import pytest

from psycopg2.errors import SyntaxError

def sqli_fuzzer(client, url, params):
    fail = False
    injections = ["'"]
    for injection in injections:
        for param in params:
            data = {k: 'foo' for k in params}
            data[param] = injection
            try:
                client.post(url, data=data)
            except SyntaxError:
                print('You seems to have an SQLi in %s for param %s' % (url, param))
                fail = True

    if fail:
        pytest.fail('Seems you are vulnerable to SQLi attacks')
```

After running `pytest --tb=short` we get:
```text
============================= test session starts ==============================
platform linux -- Python 3.5.3, pytest-5.0.1, py-1.8.0, pluggy-0.12.0
rootdir: {...}
collected 1 item

tests/test_listings.py F                                                 [100%]

=================================== FAILURES ===================================
_________________________________ test_create __________________________________
tests/test_listings.py:6: in test_create
    sqli_fuzzer(client, '/listings/create', ['title', 'description'])
tests/helpers/sqlifuzzer.py:19: in sqli_fuzzer
    pytest.fail('Seems you are vulnerable to SQLi attacks')
E   Failed: Seems you are vulnerable to SQLi attacks
----------------------------- Captured stdout call -----------------------------
INSERT INTO listings (title, description) VALUES (''', 'foo')
You seems to have an SQLi in /listings/create for param title
INSERT INTO listings (title, description) VALUES ('foo', ''')
You seems to have an SQLi in /listings/create for param description
=========================== 1 failed in 0.32 seconds ===========================

```
### Fix
Given that we have seen that the way this injection works is by breaking out of the `'`'s, we can use PostgreSQL escaping `E'\''`. For that we change our SQL query and replace every occurrence of `'` with `\'`:
```python
        sql = "INSERT INTO listings (title, description) VALUES (E'%s', E'%s')" % (
            title.replace("'", "\\'"), description.replace("'", "\\'")
        )
```

With that our test now pass:
```text
(venv) > $ pytest --tb=short
================================================================================================== test session starts ===================================================================================================
platform linux -- Python 3.5.3, pytest-5.0.0, py-1.8.0, pluggy-0.12.0
rootdir: {...}
collected 1 item
tests/test_listings.py .                                                                                                                                                                                           [100%]
================================================================================================ 1 passed in 0.95 seconds ================================================================================================
```

But this is not sufficient, if we modify our payload to be `injection\', (select version()))-- -` our query will end up being:
```sql
INSERT INTO listings (title, description) VALUES (E'injection\\', (select version()))-- -', E'\'')
```
and attacker will still be able to exploit our app.

### Testing part 2
We could keep adding more cases to our fuzzer, or use external tools, like [sqlmap](http://sqlmap.org/), which are going to be limited by the test cases we can pass to them, we could also use a Static Application Security Testing, like [bandit](https://github.com/PyCQA/bandit/).

```text
(venv) > $ bandit marketplace/**/*.py
Test results:
>> Issue: [B608:hardcoded_sql_expressions] Possible SQL injection vector through string-based query construction.
   Severity: Medium   Confidence: Low
   Location: marketplace/listings.py:27
   More Info: https://bandit.readthedocs.io/en/latest/plugins/b608_hardcoded_sql_expressions.html
26	
27	        sql = "INSERT INTO listings (title, description) VALUES (E'%s', E'%s')" % (
28	            title.replace("'", "\\'"), description.replace("'", "\\'")
29	        )

--------------------------------------------------

Code scanned:
	Total lines of code: 28
	Total lines skipped (#nosec): 0

Run metrics:
	Total issues (by severity):
		Undefined: 0.0
		Low: 0.0
		Medium: 1.0
		High: 0.0
	Total issues (by confidence):
		Undefined: 0.0
		Low: 1.0
		Medium: 0.0
		High: 0.0
Files skipped (0):
```
As we can see, the tool doesn't like our sanitization strategies and flags our code as a possible source of SQL injection.

### Fix part 2
In order to fix the SQL injetion once and for all, we should rely on prepared statements, and let the DB engine do the param sanitization, like this:
```python
        sql = "INSERT INTO listings (title, description) VALUES (%s, %s)"
        cur.execute(sql, (title, description))
```

Now both our unit test and bandit are happy!

## Description
Welcome to the Secure coding with python course. In this repository you will find a series of branches for each step of the development of a sample marketplace application. In such a development, we will be making security mistakes and introducing vulnerabilities, we will add tests for them and finally fixing them.

The branches will have the following naming scheme for easier navigation: {Chapter number}-{Chapter Name}/{code|test|fix}. I encourage you to follow the chapters in order, but you can also skip to the specific one you wish to review. 

For this course we will be using Python3, Flask and PostgreSQL.

**Proceed to [next section](https://github.com/nxvl/secure-coding-with-python/tree/2.3-sql-injection/fix3)**

## Index
### 1. Vulnerable Components
* [1-vulnerable-components/code](https://github.com/nxvl/secure-coding-with-python/tree/1-vulnerable-components/code) 
* [1-vulnerable-components/test](https://github.com/nxvl/secure-coding-with-python/tree/1-vulnerable-components/test)
* [1-vulnerable-components/fix](https://github.com/nxvl/secure-coding-with-python/tree/1-vulnerable-components/fix)

### 2. SQL Injection
* [2.1-sql-injection/code](https://github.com/nxvl/secure-coding-with-python/tree/2.1-sql-injection/code) 
* [2.1-sql-injection/test](https://github.com/nxvl/secure-coding-with-python/tree/2.1-sql-injection/test)
* [2.1-sql-injection/fix](https://github.com/nxvl/secure-coding-with-python/tree/2.1-sql-injection/fix)
* [2.2-sql-injection/test](https://github.com/nxvl/secure-coding-with-python/tree/2.2-sql-injection/test2)
* [2.2-sql-injection/fix](https://github.com/nxvl/secure-coding-with-python/tree/2.2-sql-injection/fix2)
* [2.3-sql-injection/fix](https://github.com/nxvl/secure-coding-with-python/tree/2.3-sql-injection/fix3)

### 3. Weak password storage
* [3.1-weak-password-storage/code](https://github.com/nxvl/secure-coding-with-python/tree/3.1-weak-password-storage/code) 
* [3.1-weak-password-storage/fix](https://github.com/nxvl/secure-coding-with-python/tree/3.1-weak-password-storage/fix)
* [3.2-weak-password-storage/test](https://github.com/nxvl/secure-coding-with-python/tree/3.2-weak-password-storage/test)
* [3.2-weak-password-storage/fix](https://github.com/nxvl/secure-coding-with-python/tree/3.2-weak-password-storage/fix)

### 4. Weak account secrets
* [4-weak-account-secrets/code](https://github.com/nxvl/secure-coding-with-python/tree/4-weak-account-secrets/code) 
* [4-weak-account-secrets/test](https://github.com/nxvl/secure-coding-with-python/tree/4-weak-account-secrets/test)
* [4-weak-account-secrets/fix](https://github.com/nxvl/secure-coding-with-python/tree/4-weak-account-secrets/fix)

### 5. Broken Authentication
* [5.1-broken-authentication/code](https://github.com/nxvl/secure-coding-with-python/tree/5.1-broken-authentication/code) 
* [5.1-broken-authentication/test](https://github.com/nxvl/secure-coding-with-python/tree/5.1-broken-authentication/test)
* [5.1-broken-authentication/fix](https://github.com/nxvl/secure-coding-with-python/tree/5.1-broken-authentication/fix)
* [5.2-broken-authentication/test](https://github.com/nxvl/secure-coding-with-python/tree/5.2-broken-authentication/test)
* [5.2-broken-authentication/fix](https://github.com/nxvl/secure-coding-with-python/tree/5.2-broken-authentication/fix)

### 6. Broken Deauthentication
* [6.1-broken-deauthentication/code](https://github.com/nxvl/secure-coding-with-python/tree/6.1-broken-deauthentication/code) 
* [6.1-broken-deauthentication/test](https://github.com/nxvl/secure-coding-with-python/tree/6.1-broken-deauthentication/test)
* [6.1-broken-deauthentication/fix](https://github.com/nxvl/secure-coding-with-python/tree/6.1-broken-deauthentication/fix)
* [6.2-broken-deauthentication/code](https://github.com/nxvl/secure-coding-with-python/tree/6.2-broken-deauthentication/code) 
* [6.2-broken-deauthentication/test](https://github.com/nxvl/secure-coding-with-python/tree/6.2-broken-deauthentication/test)
* [6.2-broken-deauthentication/fix](https://github.com/nxvl/secure-coding-with-python/tree/6.2-broken-deauthentication/fix)
* [6.3-broken-deauthentication/code](https://github.com/nxvl/secure-coding-with-python/tree/6.3-broken-deauthentication/code) 
* [6.3-broken-deauthentication/test](https://github.com/nxvl/secure-coding-with-python/tree/6.3-broken-deauthentication/test)
* [6.3-broken-deauthentication/fix](https://github.com/nxvl/secure-coding-with-python/tree/6.3-broken-deauthentication/fix)

### 7. Cross-Site Scripting (xss)
* [7-xss/code](https://github.com/nxvl/secure-coding-with-python/tree/7-xss/code) 
* [7-xss/test](https://github.com/nxvl/secure-coding-with-python/tree/7-xss/test)
* [7-xss/fix](https://github.com/nxvl/secure-coding-with-python/tree/7-xss/fix)

### 8. Broken Access Control
* [8-broken-access-control/code](https://github.com/nxvl/secure-coding-with-python/tree/8-broken-access-control/code) 
* [8-broken-access-control/test](https://github.com/nxvl/secure-coding-with-python/tree/8-broken-access-control/test)
* [8-broken-access-control/fix](https://github.com/nxvl/secure-coding-with-python/tree/8-broken-access-control/fix)

### 9. XML External Entities (XXE)
* [9-xxe/code](https://github.com/nxvl/secure-coding-with-python/tree/9-xxe/code) 
* [9-xxe/test](https://github.com/nxvl/secure-coding-with-python/tree/9-xxe/test)
* [9-xxe/fix](https://github.com/nxvl/secure-coding-with-python/tree/9-xxe/fix)

### 10. Sensitive Data Exposure
* [10-sensitive-data-exposure/code](https://github.com/nxvl/secure-coding-with-python/tree/10-sensitive-data-exposure/code) 
* [10-sensitive-data-exposure/test](https://github.com/nxvl/secure-coding-with-python/tree/10-sensitive-data-exposure/test)
* [10-sensitive-data-exposure/fix](https://github.com/nxvl/secure-coding-with-python/tree/10-sensitive-data-exposure/fix)
