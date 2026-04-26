---
title: "PicoCTF - Crack the Gate 2"
author: r2cook
categories:
  - PicoCTF
tags:
  - nodejs
  - expressjs
  - burp-pitchfork-attack
  - rate-limit-bypass
  - header-spoofing
  - X-Forwarded-For
render_with_liquid: false
media_subpath: /images/picoctf/crack_the_gate_2/
image:
  path: room_card.png
---

## Crack the Gate 2

Platform: picoCTF  
Category: Web Exploitation  
Difficulty: Medium 

---

## Description

The login system has been upgraded with a basic **rate-limiting mechanism** that locks out repeated failed attempts from the same source.

We received a tip that the system might still **trust user-controlled headers**.

Your objective is to bypass the rate-limiting restriction and log in using the known email address:

**ctf-player@picoctf.org**

Once logged in, retrieve the hidden secret.

- Challenge URL: [http://amiable-citadel.picoctf.net:62920/](http://amiable-citadel.picoctf.net:62920/)
    
- Password list:  
    [https://challenge-files.picoctf.net/c_amiable_citadel/746f8b0aa96c3b87d65ff8a61b25af413ae301599fe6a3434128926295901ba5/passwords.txt](https://challenge-files.picoctf.net/c_amiable_citadel/746f8b0aa96c3b87d65ff8a61b25af413ae301599fe6a3434128926295901ba5/passwords.txt)
    

---

## Given Files / Access

- Website: [http://amiable-citadel.picoctf.net:62920/](http://amiable-citadel.picoctf.net:62920/)
    
- Password list: `passwords.txt`
    

---

## Initial Recon / Enumeration

I started by intercepting the login request using **Burp Suite**.

Captured request:

```http
POST /login HTTP/1.1  
Host: amiable-citadel.picoctf.net:63772  
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:140.0) Gecko/20100101 Firefox/140.0  
Accept: */*  
Accept-Language: en-US,en;q=0.5  
Accept-Encoding: gzip, deflate, br  
Referer: http://amiable-citadel.picoctf.net:53224/  
Content-Type: application/json  
Content-Length: 56  
Origin: http://amiable-citadel.picoctf.net:53224  
Connection: keep-alive  
Priority: u=0  
  
{"email":"ctf-player@picoctf.org","password":"password"}
```

Server response:

```http
HTTP/1.1 200 OK  
X-Powered-By: Express  
Content-Type: application/json; charset=utf-8  
Content-Length: 17  
ETag: W/"11-UIVUdQWNarX1D9mk06okyEMbpS8"  
  
{"success":false}
```
From the headers we can observe:

- Backend: **Node.js Express**
    
- JSON API authentication
    
- Login attempts return a boolean success value.
    

After multiple failed attempts, the server returns:

```http
429 Too Many Requests
```

indicating that **rate limiting** is in place.

---

## Analysis

The challenge hints suggested investigating the **X-Forwarded-For header**.

The **X-Forwarded-For** header is commonly used when requests pass through:

- proxies
    
- load balancers
    
- CDNs
    

It tells the backend server the **original client IP address**.

Example:

```http
X-Forwarded-For: 1.2.3.4
```

If the backend **blindly trusts this header**, an attacker can spoof different IP addresses.

Since the rate limiter likely tracks **login attempts per IP**, rotating fake IP addresses could bypass the restriction.

---

## Exploitation / Solution

To bypass the rate limit, I added the `X-Forwarded-For` header to the request and rotated its value.

Example request:

```http
POST /login HTTP/1.1  
Host: amiable-citadel.picoctf.net  
X-Forwarded-For: 1.1.1.20  
Content-Type: application/json  
  
{"email":"ctf-player@picoctf.org","password":"password"}
```

To automate the brute force attack, I used **Burp Intruder**.

Attack configuration:

- **Attack type:** Pitchfork
    
- **Payload set 1:** rotating IP addresses

- **Payload set 2:** password list
    
    

This allowed each login attempt to use a **different IP address**, bypassing the rate-limit protection.

Intruder setup:

- Payload position for password
    
- Payload position for `X-Forwarded-For`
    

Each request looked like:

```http
X-Forwarded-For: 1.1.1.X  
password: <password_from_list>
```

This effectively avoided triggering the rate limiter.

After running the attack, one response had a **different response size**, indicating a successful login.

Inside the response we retrieved the flag.

![](4_1.png){: width="800" height="400" .shadow }

![](4_2.png){: width="800" height="400" .shadow }

![](4_3.png){: width="800" height="400" .shadow }

---

## Flag

```css
picoCTF{xff_byp4ss_brut3_3b2dc94d}
```

---

## Key Idea

The rate-limiting system trusted the **X-Forwarded-For header**, allowing attackers to spoof different IP addresses.

By rotating fake IPs during brute forcing, we bypassed the rate-limiting protection.

---

## Techniques

- Rate-limit bypass
    
- Header spoofing
    
- Credential brute forcing
    
- IP rotation
    
- Web authentication testing
    

---

## Tools Used

- Burp Suite
    
- Burp Intruder
    
- Password wordlist
    

---

## Lessons Learned

- Applications should **never trust client-controlled headers** like `X-Forwarded-For` unless properly validated.
    
- Rate limiting should rely on **trusted proxy headers** or server-side IP detection.
    
- Misconfigured proxies can lead to **rate-limit bypass vulnerabilities**.
    
- Response size differences are useful indicators when brute forcing authentication.
    

---
