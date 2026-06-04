## Summary

Cap is an Easy Linux machine hosting a network security dashboard. An **IDOR**
in the capture download feature exposes other users' PCAP files, one of which
contains plaintext FTP credentials. Those credentials grant SSH access, and a
`cap_setuid` capability set on the Python binary is abused to escalate to
**root**.

## Machine Information

| Name | Difficulty | OS | Platform |
| --- | --- | --- | --- |
| Cap | Easy | Linux | Hack The Box |

## Attack Path

1. Nmap reveals FTP, SSH and HTTP services.
2. A web dashboard allows downloading PCAP captures.
3. An IDOR exposes other users' capture files.
4. PCAP analysis reveals plaintext FTP credentials.
5. SSH access is obtained as the user `nathan`.
6. Enumeration finds the Python binary with the `cap_setuid` capability.
7. The capability is abused to escalate to root.

## Reconnaissance

Initial service enumeration was performed with **Nmap**.

```bash
sudo nmap -sV -sC -A 10.129.10.156
```

| Port | Service | Version |
| --- | --- | --- |
| 21 | FTP | vsftpd 3.0.3 |
| 22 | SSH | OpenSSH 8.2p1 |
| 80 | HTTP | Gunicorn |

The HTTP service hosted a **Security Dashboard** web application.

## Web Enumeration

The dashboard displayed network traffic statistics and allowed users to
download **PCAP files** of captured traffic. The download URL used a sequential
numeric parameter:

```
http://10.129.10.156/data/5
```

Changing the ID to another value exposed other users' captures
(`/data/0`), confirming an **Insecure Direct Object Reference (IDOR)**.

## PCAP Analysis

The capture at `/data/0` was downloaded and opened in **Wireshark**. The FTP
traffic inside contained plaintext credentials.

```
nathan : Buck3tH4TF0RM3!
```

## Initial Access (User)

The recovered credentials were reused to authenticate over SSH as `nathan`.

```bash
ssh nathan@10.129.10.156
```

This provided the initial foothold on the system. The user flag lives at
`/home/nathan/user.txt`.

## Privilege Escalation

### Enumeration

`linPEAS` was run to look for escalation vectors and flagged an interesting
capability on the Python binary.

```bash
scp linpeas.sh nathan@10.129.10.156:/tmp/
```

```
/usr/bin/python3.8 = cap_setuid,cap_net_bind_service+eip
```

### Abusing cap_setuid

The `cap_setuid` capability allows the process to change its effective UID, so
Python can be used to set UID 0 and spawn a root shell.

```bash
/usr/bin/python3.8 -c 'import os; os.setuid(0); os.system("/bin/bash")'
```

This successfully spawned a shell as root. The root flag lives at
`/root/root.txt`.

## Vulnerability Analysis

**Insecure Direct Object Reference (IDOR)** — the dashboard served PCAP captures
by a sequential numeric ID (`/data/5`), and changing the ID (`/data/0`) returned
other users' captures, disclosing plaintext credentials. Fix: enforce per-object
authorization checks server-side and use non-sequential, unguessable identifiers.

**Insecure Linux capability (`cap_setuid`)** — the Python binary carried
`cap_setuid`, letting any user running it set UID 0 and spawn a root shell. Fix:
remove unnecessary file capabilities and never grant `cap_setuid` to a
general-purpose interpreter.

## Tools Used

- Nmap
- Wireshark
- SSH
- linPEAS
- Python

## Key Takeaways

- IDOR vulnerabilities can expose sensitive internal data such as packet captures.
- Packet captures frequently contain plaintext credentials.
- Linux capabilities like `cap_setuid` are a dangerous and often overlooked privesc vector.
- Proper privilege and file-permission management is critical.
