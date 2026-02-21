---
title: Soulmate (R) - HTB
date: 2026-01-25
categories: [WriteUps, HTB]
tags: [HTB, linux, web, privesc]
image:
  path: /assets/img/posts/htb/soulmate/soulmate_full.png
---


# Overview
Soulmate is a machine of HackTheBox.
> Level: easy

> OS: linux

# Scan
```bash
sudo nmap -n -sV -sS -Pn -p- -oN scan.txt 10.129.238.52 
Starting Nmap 7.95SVN ( https://nmap.org ) at 2026-01-25 10:46 EST
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 10.0p2 Debian 8 (protocol 2.0)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

sudo nmap -sU -p- -T4 --min-rate 1000 10.129.238.52 -oN scan-udp.txt
Starting Nmap 7.95SVN ( https://nmap.org ) at 2026-01-25 11:28 EST
PORT    STATE SERVICE
500/udp open  isakmp

sudo nmap -sU -sC -sV -p500 10.129.238.52
Starting Nmap 7.95SVN ( https://nmap.org ) at 2026-01-25 11:44 EST
PORT    STATE SERVICE VERSION
500/udp open  isakmp?
| ike-version: 
|   attributes: 
|     XAUTH
|_    Dead Peer Detection v1.0
```


**The machine is still active so this post is not yet available!**