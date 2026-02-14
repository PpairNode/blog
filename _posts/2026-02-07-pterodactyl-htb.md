---
title: Pterodactyl (R) - HTB
date: 2026-02-07
categories: [WriteUps, HTB]
tags: [HTB, linux, web, privesc]
image:
  path: /assets/img/posts/htb/pterodactyl/pterodactyl_full.png
---


# Overview
Pterodactyl is the 2nd machine of HackTheBox Season 10.
> Level: medium
> OS: linux

# Scan
## Nmap
First scanning all ports with a rapid SYN scan:
```bash
sudo nmap -n -sS -Pn -p- -oN scan.txt 10.129.2.195
PORT     STATE    SERVICE
22/tcp open ssh
80/tcp open http
```

- Add to `/etc/hosts` our target
```bash
echo '<ip> pterodactyl.htb' | sudo tee /etc/hosts -a
```

**The machine is still active so this post is not yet available!**