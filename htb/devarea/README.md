# Hack The Box — DevArea: Exploitation Summary

## Overview

This is a medium-difficulty Linux machine built around a Java SOAP service and a
stack of internal automation tooling (Hoverfly API simulator and a custom
"SysWatch" monitoring suite). The foothold is an Apache CXF file-read primitive
delivered over MTOM/XOP; privilege escalation chains a forged Flask session, a
command-injection web GUI, and a symlink-following root log viewer.

## Critical Vulnerabilities Exploited

**1. Apache CXF MTOM/XOP File Read (CVE-2022-46363/46364)**

Anonymous FTP leaks `employee-service.jar`, a small Apache CXF 3.2.14 JAX-WS SOAP
service (Aegis databinding) whose `content` field is echoed in responses. Classic
SOAP-envelope XXE is blocked, but an MTOM/XOP `xop:Include href="file://..."` is
resolved server-side and the bytes are returned base64-encoded — arbitrary file
read as the `dev_ryan` service account, with no egress required.

**2. Credentials Leaked in a World-Readable systemd Unit**

The file read recovers `/etc/systemd/system/hoverfly.service`, exposing the
Hoverfly admin/proxy credentials (`admin:O7IJ27MyyXiU`) passed on the command line.

**3. Hoverfly Middleware Command Execution**

Hoverfly executes a local binary/script ("middleware") for each proxied request.
Using the leaked credentials and a JWT from `/api/token-auth`, the attacker
registers middleware, switches the proxy to `modify` mode, and triggers it with a
proxied request — running arbitrary commands as `dev_ryan` and planting an SSH key
for the user shell.

**4. Forged Flask Session + `shell=True` Command Injection**

`dev_ryan` may run `/opt/syswatch/syswatch.sh` as root, but the interesting bug is
the SysWatch Flask GUI (localhost:7777, running as `syswatch`). Its admin password
was seeded differently from the leaked default, so login is bypassed by **forging
a session cookie** with the world-readable `SYSWATCH_SECRET_KEY`. The
`/service-status` endpoint then builds a `subprocess.run(..., shell=True)` command
from user input behind a denylist regex — bypassed with a pipe and octal `printf`
to rebuild blocked characters — giving code execution as `syswatch`.

**5. `sudo` Double-Symlink Arbitrary Root Read**

`syswatch.sh logs <file>` runs as root and tries to be symlink-safe, but it only
validates the *first* symlink hop's (slash-free) target name, then `cat`s
`$LOG_DIR/$target`. Because `cat` follows a *second* symlink anywhere, `syswatch`
(who owns the log directory) plants a two-hop chain pointing at `/root/root.txt`
(or `/etc/shadow`), and the root sudo rule reads it.

## Exploitation Chain

```
Nmap → anon FTP (employee-service.jar) → decompile (Apache CXF 3.2.14) →
MTOM/XOP file read as dev_ryan → leak hoverfly.service creds →
Hoverfly admin API (JWT) → middleware RCE → SSH key → dev_ryan (user flag) →
leak SYSWATCH_SECRET_KEY → forge Flask cookie → /service-status cmd injection →
syswatch → plant double-symlink in logs → sudo syswatch.sh logs → root flag
```

## Key Lessons

A leaked build artifact is a free source map — decompile before guessing. When
SOAP-envelope XXE is patched, MTOM/XOP can still reach the same parser, and a
reflected field becomes a no-egress file-read oracle. Signing keys are
credentials: a leaked Flask `secret_key` ends authentication outright. Denylist
input filters fall to encoding (pipes, command substitution, octal), and symlink
checks must canonicalize the full path — validating one hop while `cat` follows
the rest is no protection at all.
