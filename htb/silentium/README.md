## Summary

Silentium is an Easy Linux machine solved through chaining rather than a single
exploit. Virtual host fuzzing reveals a `staging` subdomain whose
`forgot-password` API leaks a temporary reset token, enabling an **account
takeover** and SSH access as `ben`. Local enumeration of `/proc/1/environ`
exposes secrets and an internal service on port 3001, reached via SSH tunneling:
an internal **Gogs** instance, exploited through a symlink hook injection to
escalate to **root**.

## Machine Information

| Name | Difficulty | OS | Platform |
| --- | --- | --- | --- |
| Silentium | Easy | Linux | Hack The Box |

## Attack Path

1. Nmap reveals SSH and HTTP.
2. Virtual host fuzzing discovers `staging.silentium.htb`.
3. The staging API leaks a password-reset token via `forgot-password`.
4. The token is used to reset `ben`'s password (account takeover).
5. SSH access is obtained as `ben`.
6. `/proc/1/environ` exposes secrets and an internal service on port 3001.
7. SSH tunneling reaches the internal Gogs instance.
8. A Gogs symlink hook injection escalates to root.

## Reconnaissance

Initial enumeration was performed with **Nmap**.

```bash
nmap -sC -sV -A -T4 10.129.27.123
```

Only SSH (22) and HTTP (80) were open, with a redirect to `http://silentium.htb/`
— a strong hint of virtual host routing and possible hidden subdomains.

## Web Enumeration

After adding the host to `/etc/hosts`, the main application was a clean
corporate site with no obvious functionality, suggesting the real surface lay in
a subdomain or API.

## Subdomain Discovery

Virtual host fuzzing was performed with `ffuf` against the `Host` header.

```bash
ffuf -u http://silentium.htb/ \
  -H "Host: FUZZ.silentium.htb" \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
  -fs 8753 -mc 200
```

This revealed `staging.silentium.htb` — typically a less hardened environment
and a good candidate for logic flaws.

## API Enumeration

The staging environment exposed an authentication interface and, more
importantly, a password-recovery endpoint.

```bash
curl -s -X POST http://staging.silentium.htb/api/v1/account/forgot-password \
  -H "Content-Type: application/json" \
  -d '{"user":{"email":"ben@silentium.htb"}}'
```

Instead of a generic response, the API returned sensitive account data,
including a `tempToken`. Leaking the reset token to the client breaks the entire
recovery mechanism and enables account takeover.

## Exploitation — Password Reset Abuse

With the leaked `tempToken`, the reset endpoint accepted a new password for the
target account.

```bash
curl -i -X POST http://staging.silentium.htb/api/v1/account/reset-password \
  -H "Content-Type: application/json" \
  -d '{
    "user": {
      "email": "ben@silentium.htb",
      "tempToken": "EFSKdWfe8hE1MrLobGKklbHsF8xDTZMafc809hCVpg8SZxgTxf...",
      "password": "NewPass123!",
      "confirmPassword": "NewPass123!"
    }
  }'
```

The password change succeeded — a pure business-logic flaw turning user
enumeration into full account control.

## Initial Access (User)

With the new credentials, SSH access was obtained as `ben`.

```bash
ssh ben@10.129.23.208
```

The user flag lives at `/home/ben/user.txt`.

## Privilege Escalation

### Enumeration

Reading the init process environment exposed secrets and pointed at an internal
service.

```bash
cat /proc/1/environ
```

It revealed values such as `SMTP_PASSWORD`, `JWT_SECRET`,
`JWT_REFRESH_TOKEN_SECRET`, Flowise parameters, and an internal service
listening on port 3001.

### Pivoting to the internal Gogs

An SSH tunnel exposed the internal service locally.

```bash
ssh -L 3001:127.0.0.1:3001 ben@10.129.25.77
```

The service on port 3001 was an internal **Gogs** (self-hosted Git) instance.

### Gogs symlink hook injection

Gogs was exploited via a symlink hook injection chain: authenticate, create a
controlled repository, inject a malicious structure with a symlink, and trigger
a Git hook that writes/executes a payload on the host. The exploit produced a
SUID artifact and a shell with `euid=0`, confirming root. The root flag lives at
`/root/root.txt`.

## Vulnerability Analysis

**Information disclosure in password recovery** — `forgot-password` returned
internal account data and a reset token that should never reach the client. Fix:
never return reset tokens in API responses; deliver them only out-of-band (e.g.
email).

**Password reset logic flaw** — the reset endpoint accepted the leaked token as
sufficient proof, allowing another user's password to be changed. Fix: bind
tokens to a single account, expire them quickly, and require server-side
validation that the requester owns the token.

**Secrets exposed in environment variables** — credentials and keys were readable
in `/proc/1/environ`, enabling internal architecture mapping. Fix: avoid passing
secrets via environment variables for long-lived processes; use a secrets
manager and restrict `/proc` access.

**Privileged internal Gogs** — the internal Gogs instance was exploitable via
symlink hook injection to reach root. Fix: upgrade Gogs, disable custom Git
hooks for untrusted users, and run it with least privilege.

## Tools Used

- Nmap
- ffuf
- curl
- SSH
- Public Gogs exploit / PoC

## Key Takeaways

- Staging environments are often the least hardened part of the infrastructure.
- Business-logic flaws can be more dangerous than classic memory or injection bugs.
- A user foothold mainly serves to reveal the real internal architecture.
- Services bound to localhost are not safe once an attacker has a shell and can tunnel.
