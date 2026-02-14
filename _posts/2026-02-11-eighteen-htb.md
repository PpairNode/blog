---
title: Eighteen (U) - HTB
date: 2026-02-11
categories: [WriteUps, HTB]
tags: [HTB, windows, AD, privesc]
image:
  path: /assets/img/posts/htb/eighteen/eighteen_full.png
---


# Overview
Eighteen is the 9th machine of HackTheBox Season 9.
> Level: easy
> OS: windows

Machine Information
As is common in real life Windows penetration tests, you will start the Eighteen box with credentials for the following account: kevin / iNa2we6haRj2gaw!

# Scan
- Full scan
```bash
sudo nmap -n -sS -Pn -p- -oN scan.txt 10.129.6.13

PORT     STATE SERVICE
80/tcp   open  http
1433/tcp open  ms-sql-s
5985/tcp open  wsman

Nmap done: 1 IP address (1 host up) scanned in 600.23 seconds
```
- Rapid scan windows
```bash
sudo nmap -n -sS -Pn -p21,22,53,80,88,135,139,143,443,445,1433,1521,3128,3306,3389,8080,8443 -oN win_scan.txt --open 10.129.6.13

PORT     STATE SERVICE
80/tcp   open  http
1433/tcp open  ms-sql-s
```

- Scpecialized scan
```bash
nmap -n -sV -sC -Pn -p80,1433,5985 10.129.6.13
Starting Nmap 7.95SVN ( https://nmap.org ) at 2026-02-11 05:17 EST
Nmap scan report for 10.129.6.13
Host is up (0.11s latency).

PORT     STATE SERVICE  VERSION
80/tcp   open  http     Microsoft IIS httpd 10.0
|_http-server-header: Microsoft-IIS/10.0
|_http-title: Did not follow redirect to http://eighteen.htb/
1433/tcp open  ms-sql-s Microsoft SQL Server 2022 16.00.1000.00; RTM
|_ssl-date: 2026-02-11T17:17:51+00:00; +6h59m54s from scanner time.
| ms-sql-ntlm-info: 
|   10.129.6.13:1433: 
|     Target_Name: EIGHTEEN
|     NetBIOS_Domain_Name: EIGHTEEN
|     NetBIOS_Computer_Name: DC01
|     DNS_Domain_Name: eighteen.htb
|     DNS_Computer_Name: DC01.eighteen.htb
|     DNS_Tree_Name: eighteen.htb
|_    Product_Version: 10.0.26100
| ms-sql-info: 
|   10.129.6.13:1433: 
|     Version: 
|       name: Microsoft SQL Server 2022 RTM
|       number: 16.00.1000.00
|       Product: Microsoft SQL Server 2022
|       Service pack level: RTM
|       Post-SP patches applied: false
|_    TCP port: 1433
| ssl-cert: Subject: commonName=SSL_Self_Signed_Fallback
| Not valid before: 2026-02-11T15:49:16
|_Not valid after:  2056-02-11T15:49:16
5985/tcp open  http     Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: mean: 6h59m53s, deviation: 0s, median: 6h59m53s

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 16.09 seconds
```

We are against a `Domain Controller` machine.


- Add to `/etc/hosts` our target
```bash
echo '<ip> eighteen.htb DC01.eighteen.htb' | sudo tee /etc/hosts -a
```

**The machine is still active so this post is not yet available!**