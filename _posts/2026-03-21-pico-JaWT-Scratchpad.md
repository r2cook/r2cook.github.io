---
title: "PicoCTF - JaWT Scratchpad"
author: r2cook
categories:
  - PicoCTF
tags:
  - jwt
  - jwt-manipulation
  - john
  - hashcat
  - jwt.io
render_with_liquid: false
media_subpath: /images/picoctf/JaWT-Scratchpad/
image:
  path: room_card.png
---

#  JaWT Scratchpad

Platform: picoCTF (or Web Challenge)  
Category: Web Exploitation  
Difficulty: Medium 

---

## Description

This challenge focuses on understanding **JSON Web Tokens (JWT)** and how improper implementation—specifically weak secret keys—can lead to authentication bypass.

The goal is to escalate privileges and gain **admin access** by manipulating the JWT.

---

## Given Files / Access

- Files: None
    
- URL: Target web application
    
- IP: N/A
    

---

## Initial Recon / Enumeration

After accessing the web application and logging in, a cookie containing a JWT was observed.

### Commands / Tools:

- Browser DevTools (F12)
    
- Cookie Editor extension
    

### Notes:

- The hint suggested inspecting cookies.
    
- The session is maintained using a JWT stored in the browser.
    

---

## Analysis

### What is happening?

The application uses JWT for authentication. A JWT consists of three parts:

![](14_1.png){: width="800" height="400" .shadow }

```
header.payload.signature
```

Decoding the token:

- **Header:**
    

```json
{"typ":"JWT","alg":"HS256"}
```

- **Payload:**
    

```json
{"user":"username"}
```

- **Signature:**  
    Used to verify integrity using a secret key.
    
![](14_2.png){: width="800" height="400" .shadow }

---

### What is the vulnerability / trick?

- The token uses **HS256 (HMAC-SHA256)**.
    
- This algorithm relies on a **shared secret key**.
    
- If the secret key is weak, it can be brute-forced.
    

📌 Vulnerability:

> Weak JWT secret allows attackers to forge valid tokens.

---

## Exploitation / Solution

Steps to exploit:

1. Extract the JWT from the cookie.
    
2. Use a tool like:
    
    - `JohnTheRipper`
        
    - or `hashcat`  
        to brute-force the secret key.
    ```shell
    $ hashcat -m 16500 jwt.txt ~/tools/rockyou.txt
    ...
    eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJ1c2VyIjoidXNlciJ9.hlht7uqMfJWBXINRilmhIEstPPs0jO_F0HgWM9LZnJc:ilovepico
    ...
    ```
1. Recover the secret key.
    
2. Modify the payload:
    

```
{: .nolineno }json
{"user":"admin"}
```

![](14_3.png){: width="800" height="400" .shadow }

5. Re-sign the token using the recovered secret.
    
6. Replace the original JWT in the browser (using Cookie Editor).

![](14_4.png){: width="800" height="400" .shadow }
    
7. Refresh the page → gain admin access ✅
    

---

## Flag

```
picoCTF{jawt_was_just_what_you_thought_bbb82bd4a57564aefb32d69dafb60583}
```

---