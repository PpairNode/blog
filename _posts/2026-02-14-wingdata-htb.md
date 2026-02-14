---
title: WingData (R) - HTB
date: 2026-02-14
categories: [WriteUps, HTB]
tags: [HTB, linux, web, privesc]
image:
  path: /assets/img/posts/htb/wingdata/wingdata_full.png
---


# Overview
WingData is the 3rd machine of HackTheBox Season 10.
> Level: easy
> OS: linux

# Scan
## Nmap
First scanning all ports with a rapid SYN scan:
```bash
sudo nmap -n -sS -Pn -p- -oN scan.txt <ip>
PORT      STATE SERVICE
22/tcp    open  ssh
80/tcp    open  http
```

- Add to `/etc/hosts` our target
```bash
echo '<ip> wingdata.htb' | sudo tee /etc/hosts -a
```

**The machine is still active so this post is not yet available!**
