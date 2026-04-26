---
title: 'TryHackMe - Wonderland'
author: r2cook
categories: [TryHackMe]
tags: [ Steganography, steghide, PrivEsc, pkexec, PwnKit]
render_with_liquid: false
media_subpath: /images/thm/Wonderland/
image:
  path: room_card.jpeg
---

> In this walkthrough, we compromise the TryHackMe **Wonderland** machine by following subtle web hints, extracting steganographic clues from an image, recursively discovering hidden directories, uncovering **Alice’s SSH credentials** from concealed HTML source code, and escalating privileges through **PwnKit (CVE-2021-4034)** to obtain both user and root flags.

## Initial Setup

Add the target to your hosts file:

```plaintext
10.130.150.114 wonderland.thm
```
{: file="/etc/hosts" }

---

## Enumeration

### Full Port Scan

```shell
/tmp/tmp.3XMfgXE3zI » nmap -p- -sV -sC -Pn --min-rate 1000 -oN full_nmap_scan.txt wonderland.thm
Starting Nmap 7.95 ( https://nmap.org ) at 2026-04-02 05:03 EET
Nmap scan report for wonderland.thm (10.130.150.114)
Host is up (0.055s latency).
Not shown: 65533 closed tcp ports (conn-refused)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   2048 8e:ee:fb:96:ce:ad:70:dd:05:a9:3b:0d:b0:71:b8:63 (RSA)
|   256 7a:92:79:44:16:4f:20:43:50:a9:a8:47:e2:c2:be:84 (ECDSA)
|_  256 00:0b:80:44:e6:3d:4b:69:47:92:2c:55:14:7e:2a:c9 (ED25519)
80/tcp open  http    Golang net/http server (Go-IPFS json-rpc or InfluxDB API)
|_http-title: Follow the white rabbit.
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
{: .nolineno }

### Key Findings

- **Port 80** gives the first major hint: **“Follow the white rabbit.”**
    
- This strongly suggests the web path and image assets contain the initial foothold clue.
    

---

## Web Enumeration & Stego Discovery

Download the rabbit image from the web root:

```shell
wget http://wonderland.thm/img/white_rabbit_1.jpg
```
{: .nolineno }

Validate the file:

```shell
file white_rabbit_1.jpg
```
{: .nolineno }

```text
white_rabbit_1.jpg: JPEG image data, JFIF standard 1.01
```

The image looks normal, so the next logical step is **steganography analysis**.

### Steghide Inspection

```shell
steghide info white_rabbit_1.jpg
```
{: .nolineno }

Output reveals embedded content:

```text
embedded file "hint.txt"
```

Extract it:

```shell
steghide extract -sf white_rabbit_1.jpg
```
{: .nolineno }

Read the clue:

```shell
cat hint.txt
```
{: .nolineno }

```text
follow the r a b b i t%
```

This confirms we need to enumerate nested directories under `/r/`.

---

## Rabbit Hole Directory Fuzzing

Start fuzzing the `/r/` path:

```shell
ffuf -u http://wonderland.thm/r/FUZZ \
  -w /usr/share/wordlists/SecLists/Discovery/Web-Content/common.txt \
  -e .php,.txt,.zip,.old,.log \
  -mc 200,204,301,302,307,401,403 \
  -fc 27 -c | tee ffuf_common_result.txt
```
{: .nolineno }

The trick here is to **keep recursively following the hint** until you reach:

```text
/r/a/b/b/i/t
```

Exactly as the stego clue suggested.

![](r_a_b_b_i_t_dir_enum.webp){: width="800" height="400" .shadow }

---

## Hidden Source Code Clue

Inside the final rabbit directory, inspect the **page source**.

![](r_a_b_b_i_t_dir_pagesource.webp){: width="800" height="400" .shadow }

A hidden HTML element reveals credentials:

```html
<p style="display: none;">alice:[REDACTED]</p>
```

### Initial Access

Use the discovered credentials over SSH:

```shell
ssh alice@wonderland.thm
```
{: .nolineno }

Login succeeds.

![](alice_ssh_login.webp){: width="800" height="400" .shadow }

---

## Privilege Escalation

After gaining a shell as `alice`, enumerate SUID binaries:

```shell
find / -perm -4000 -type f 2>/dev/null
```
{: .nolineno }

One critical binary stands out:

- `/usr/bin/pkexec`
    

Check the version:

```shell
/usr/bin/pkexec --version
```
{: .nolineno }

```text
pkexec version 0.105
```

This version is vulnerable to:

> **PwnKit — CVE-2021-4034**

A reliable local privilege escalation to root.

![](priv_esc.webp){: width="800" height="400" .shadow }

After exploitation, verify both flags:

```shell
cat /root/user.txt
cat /home/alice/root.txt
```
{: .nolineno }

---

# Flags

## User Flag

```text
[REDACTED]
```

## Root Flag

```text
[REDACTED]
```

---