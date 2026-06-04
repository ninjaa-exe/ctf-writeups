## Summary

Snapped is a Hard Linux machine. VHost fuzzing reveals an `admin` subdomain
hosting **Nginx UI**, vulnerable to an unauthenticated backup download
(CVE-2026-27944) that also leaks the data needed to decrypt it. The backup
contains a SQLite database of bcrypt hashes; cracking `jonathan`'s hash grants
SSH access. A SUID `snap-confine` is then exploited via **CVE-2026-3888**
(snap-confine / systemd-tmpfiles LPE) to gain a root shell.

## Machine Information

| Name | Difficulty | OS | Platform |
| --- | --- | --- | --- |
| Snapped | Hard | Linux | Hack The Box |

## Attack Path

1. Nmap reveals SSH and an Nginx HTTP service.
2. VHost fuzzing discovers `admin.snapped.htb`.
3. The panel is identified as Nginx UI.
4. An unauthenticated backup is downloaded (CVE-2026-27944).
5. The backup is decrypted and analyzed.
6. bcrypt hashes are extracted from the SQLite database.
7. `jonathan`'s hash is cracked.
8. SSH access is obtained as `jonathan`.
9. A SUID `snap-confine` is found during enumeration.
10. CVE-2026-3888 is exploited to create a SUID root shell.

## Reconnaissance

Initial enumeration was performed with **Nmap**.

```bash
nmap -sC -sV -A -T4 <IP>
```

| Port | Service | Notes |
| --- | --- | --- |
| 22 | SSH | OpenSSH 9.6p1 (Ubuntu) |
| 80 | HTTP | Nginx 1.24.0, redirects to `snapped.htb` |

The host was added to `/etc/hosts`.

## Web Enumeration

`http://snapped.htb` was a static corporate site with no obvious functionality.
Since it used virtual host routing, the next step was subdomain fuzzing.

## Subdomain Enumeration

```bash
ffuf -u http://snapped.htb \
  -H "Host: FUZZ.snapped.htb" \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt -mc 200
```

This revealed `admin.snapped.htb`, which was added to `/etc/hosts`. The subdomain
hosted an **Nginx UI** login panel, identified as vulnerable to
**CVE-2026-27944**.

## Exploitation — Nginx UI Unauthenticated Backup (CVE-2026-27944)

CVE-2026-27944 affects Nginx UI before 2.3.3: the `/api/backup` endpoint is
reachable without authentication and returns, in the `X-Backup-Security` header,
the material needed to decrypt the backup. An unauthenticated attacker can
download the backup and recover configs, the SQLite database, tokens and
credentials.

```bash
python poc.py --target http://admin.snapped.htb --decrypt
```

The script recovered and decrypted the backup, yielding files such as
`hash_info.txt`, `nginx-ui.zip` and `nginx.zip`.

## Backup Analysis

A search through the extracted files surfaced sensitive data:

```bash
grep -RniE "password|user|secret|token|key|jwt|sqlite|database" .
```

`app.ini` pointed at the application's SQLite database
(`/var/lib/nginx-ui/database.db`) and exposed secrets like `JwtSecret`, but the
key asset was the database itself.

The `users` table was read with `sqlite3`:

```bash
sqlite3 -header -column nginx-ui/database.db "SELECT * FROM users;"
```

Both `admin` and `jonathan` had bcrypt hashes; `jonathan`'s was saved for
cracking.

## Privilege Escalation

### Cracking the hash

```bash
john --format=bcrypt --wordlist=/usr/share/wordlists/rockyou.txt hashes.txt
```

```
jonathan : linkinpark
```

The password was reused for SSH:

```bash
ssh jonathan@<IP>
```

The user flag lives at `/home/jonathan/user.txt`.

### Enumeration

`jonathan` had no sudo rights, but enumeration found `snapd` with a SUID
`snap-confine`:

```bash
snap version          # snapd 2.63.1+24.04 on Ubuntu 24.04
ls -l /usr/lib/snapd/snap-confine
# -rwsr-xr-x 1 root root ... /usr/lib/snapd/snap-confine
```

### snap-confine LPE (CVE-2026-3888)

The `snapd` version was vulnerable to **CVE-2026-3888**, a local privilege
escalation abusing the interaction between `snap-confine` and
`systemd-tmpfiles`. The exploit wins a race condition during namespace creation
to replace the dynamic linker (`ld-linux-x86-64.so.2`) with a controlled
payload, which the SUID `snap-confine` then loads as root.

The exploit and payload were compiled and transferred:

```bash
gcc -O2 -static -o exploit exploit_suid.c
gcc -nostdlib -static -Wl,--entry=_start -o librootshell.so librootshell_suid.c
scp exploit librootshell.so jonathan@snapped.htb:/home/jonathan/
```

On the target, the exploit was run with the payload:

```bash
./exploit ./librootshell.so
```

It won the race against `snap-confine`, injected the payload into the poisoned
namespace, and created a SUID copy of bash, yielding `euid=0`:

```
uid=1000(jonathan) gid=1000(jonathan) euid=0(root) groups=1000(jonathan)
```

The root flag lives at `/root/root.txt`.

## Vulnerability Analysis

**Unauthenticated backup in Nginx UI (CVE-2026-27944)** — `/api/backup` was
reachable without authentication and exposed the material to decrypt the backup,
disclosing all configs, the database and secrets. Fix: upgrade Nginx UI,
authenticate the backup endpoint, and never return decryption material to
clients.

**Sensitive data in backup** — the backup contained a SQLite database with user
hashes and application secrets. Fix: encrypt backups with keys held separately
and restrict who can generate/download them.

**Crackable bcrypt hash** — `jonathan`'s bcrypt hash was cracked with a common
wordlist, giving SSH access. Fix: enforce strong password policies.

**Credential reuse** — the application password was also valid for the local
`jonathan` account. Fix: enforce unique credentials per service.

**Local privilege escalation (CVE-2026-3888)** — a vulnerable `snapd` allowed
abusing `snap-confine` / `systemd-tmpfiles` for root. Fix: patch snapd and keep
the OS updated.

## Tools Used

- Nmap
- ffuf
- Python 3
- sqlite3
- John the Ripper
- SSH / SCP
- GCC
- Public CVE-2026-3888 exploit

## Key Takeaways

- Enumerate virtual hosts whenever the web server redirects to a specific domain.
- Backups accessible without authentication can compromise an entire application.
- Encrypting a backup is pointless if the key or IV is handed to the attacker.
- SQLite databases in backups commonly hold credentials, tokens and configs.
- Password reuse between an app and the OS enables easy lateral movement.
- SUID binaries deserve close attention; recent local CVEs (like in `snapd`) can be decisive on modern Linux.
