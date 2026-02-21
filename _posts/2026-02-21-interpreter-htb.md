---
title: Interpreter (R) - HTB
date: 2026-02-21
categories: [WriteUps, HTB]
tags: [HTB, linux, web, privesc]
image:
  path: /assets/img/posts/htb/interpreter/interpreter_full.png
---


# Overview
Interpreter is the 4th machine of HackTheBox Season 10.
> Level: medium (feeling: easy)

> OS: linux

# Scan
## Nmap
First scanning all ports with a rapid SYN scan:
```bash
sudo nmap -n -sS -Pn -p- -oN scan.txt 10.129.3.65

# Nmap 7.95SVN scan initiated Sat Feb 21 14:03:23 2026 as: nmap -n -sS -Pn -p- -oN scan.txt 10.129.3.65
Nmap scan report for 10.129.3.65
Host is up (0.059s latency).
Not shown: 65531 closed tcp ports (reset)
PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
443/tcp  open  https
6661/tcp open  unknown
```

Adding to `/etc/hosts` our target
```bash
echo '<ip> interpreter.htb' | sudo tee /etc/hosts -a
```

**The machine is still active so this post is not yet available!**
