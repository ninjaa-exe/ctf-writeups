## Summary

UnderPass is an Easy Linux machine where the web server only serves a default
Apache page. UDP enumeration reveals **SNMP**, which leaks references to a
**daloRADIUS** deployment. The panel is accessible with default credentials,
exposing a user hash that is cracked for SSH access as `svcMosh`. A sudo rule
allowing `mosh-server` is then abused to obtain a **root** shell.

## Machine Information

| Name | Difficulty | OS | Platform |
| --- | --- | --- | --- |
| UnderPass | Easy | Linux | Hack The Box |

## Attack Path

1. Nmap reveals SSH and HTTP services.
2. The web server only shows the default Apache page.
3. UDP enumeration discovers an SNMP service.
4. SNMP leaks information about daloRADIUS.
5. The daloRADIUS panel is accessed with default credentials.
6. A password hash is extracted and cracked.
7. SSH access is obtained as `svcMosh`.
8. A sudo rule allows running `mosh-server`.
9. The Mosh session is abused to obtain a root shell.

## Reconnaissance

Initial enumeration was performed with **Nmap**.

```bash
nmap -sC -sV -A 10.10.11.48
```

| Port | Service |
| --- | --- |
| 22 | SSH |
| 80 | HTTP (Apache) |

## Web Enumeration

The web server only returned the **default Apache page**, so enumeration moved
to other protocols.

## SNMP Enumeration

A UDP scan revealed that **SNMP** was running, and `snmpwalk` returned useful
information, including references to **daloRADIUS**.

```bash
snmpwalk -v2c -c public 10.10.11.48
```

This indicated that a RADIUS management interface might be reachable on the web
server.

## Accessing daloRADIUS

The operator login was located at:

```
/daloradius/app/operators/login.php
```

The panel accepted **default credentials**:

```
administrator : radius
```

## Credential Discovery

Inside the dashboard, a user account and password hash were found. The hash was
cracked, revealing system credentials:

```
svcMosh : underwaterfriends
```

## Initial Access (User)

The recovered credentials were reused to authenticate over SSH as `svcMosh`.

```bash
ssh svcMosh@10.10.11.48
```

This provided the initial foothold on the system. The user flag lives at
`/home/svcMosh/user.txt`.

## Privilege Escalation

### Enumeration

`sudo -l` showed the user could run `mosh-server` as root without a password.

```
(ALL) NOPASSWD: /usr/bin/mosh-server
```

### Abusing mosh-server

Starting the server with sudo produced a MOSH key and port. Connecting to that
session yielded an interactive shell running as **root**.

```bash
sudo /usr/bin/mosh-server new
```

The root flag lives at `/root/root.txt`.

## Vulnerability Analysis

**SNMP information disclosure** — public SNMP access (community string `public`)
exposed internal configuration details and revealed the daloRADIUS deployment.
Fix: disable SNMP if unused, restrict it to trusted hosts, and replace default
community strings with SNMPv3 authentication.

**Default credentials** — the daloRADIUS panel accepted the shipped
`administrator:radius` login, granting access to configuration and credentials.
Fix: change default credentials on deployment and restrict access to management
panels.

**Privilege escalation via `mosh-server` (sudo)** — `svcMosh` could run
`mosh-server` as root via sudo, which was abused for a root shell. Fix: remove
unnecessary NOPASSWD sudo rules and avoid granting sudo on binaries that spawn
interactive sessions.

## Tools Used

- Nmap
- snmpwalk
- SSH
- Hash cracking tools
- Mosh

## Key Takeaways

- UDP enumeration can reveal services missed by a TCP-only scan.
- SNMP frequently leaks sensitive system information.
- Default credentials remain a common, high-impact issue.
- Misconfigured sudo rules are a reliable privilege escalation vector.
