## Summary

Olympus is a Medium Linux machine. A **SQL injection** in `category.php` exposes
the application database, leaking user hashes and internal chat messages that
point to a separate chat subdomain with an insecure **file-upload** feature. A
cracked password is reused to log in to the chat, and a PHP web shell is uploaded
to gain code execution as `www-data`. A misused `cputils` binary copies `zeus`'s
SSH private key, whose passphrase is cracked to obtain a stable shell. A hidden,
privileged backdoor binary is finally abused to escalate to **root**.

## Machine Information

| Name | Difficulty | OS | Platform |
| --- | --- | --- | --- |
| Olympus | Medium | Linux | TryHackMe |

## Attack Path

1. Nmap reveals SSH and HTTP on the main host.
2. Feroxbuster discovers the `~webmaster` application.
3. `category.php?cat_id=` is vulnerable to SQL injection.
4. sqlmap dumps users, hashes and internal chat messages.
5. A user hash is cracked and reused on the `chat.olympus.thm` subdomain.
6. An insecure file upload is abused to gain a shell as `www-data`.
7. The `cputils` binary is abused to copy `zeus`'s SSH private key.
8. The key passphrase is cracked, granting stable SSH access as `zeus`.
9. A hidden backdoor invokes a privileged binary used to escalate to root.

## Reconnaissance

Initial service enumeration was performed with **Nmap**.

```bash
nmap -sC -sV -A -T4 10.65.146.182
```

| Port | Service |
| --- | --- |
| 22 | SSH |
| 80 | HTTP |

With a small attack surface, the web service was the primary candidate for
deeper enumeration.

## Web Enumeration

Content discovery was run against the HTTP service.

```bash
feroxbuster -u http://olympus.thm -w /usr/share/seclists/Discovery/Web-Content/big.txt -s 200
```

Enumeration revealed a PHP application under `~webmaster`, including
`category.php`, `includes/db.php` and `login.php`. The numeric parameter
`category.php?cat_id=` stood out as a likely SQL injection candidate.

## SQL Injection

The `cat_id` parameter was tested and confirmed vulnerable with **sqlmap**, which
enumerated the `olympus` database.

```bash
sqlmap -u "http://olympus.thm/~webmaster/category.php?cat_id=1" \
  -p cat_id -D olympus --tables --batch
```

The database exposed the tables `categories`, `chats`, `comments`, `flag`,
`posts` and `users`.

## Database Looting

Dumping the `users` table revealed application accounts (**prometheus**, **zeus**,
**root**) with bcrypt password hashes. The `chats` table was more valuable,
referencing `prometheus_password.txt`, an upload directory, the fact that
uploaded files were renamed, and a recent upload of `shell.php`. This strongly
suggested a separate chat application with a file-upload feature that could lead
to RCE.

The `flag` table also held an in-database flag (omitted here).

## Credential Cracking

The bcrypt hashes were cracked offline with **John the Ripper**.

```bash
john --show hashes.txt
```

This recovered the credential `prometheus:summertime`, a strong candidate for
reuse on the chat application.

## Chat Subdomain

Enumeration of the chat subdomain exposed a `/uploads/` directory and the chat
login.

```bash
feroxbuster -u http://chat.olympus.thm -w /usr/share/seclists/Discovery/Web-Content/big.txt -s 200
```

The cracked credentials authenticated to `chat.olympus.thm`, confirming the
upload folder, randomized filenames and recent `.php` attachments seen in the
database.

## File Upload to RCE

The upload feature was abused to host a PHP payload in a web-interpreted
directory. A listener was prepared on the attacker machine.

```bash
nc -vnlp 1337
```

Triggering the uploaded file returned a shell as `www-data` — the pivot point
from web exploitation to local privilege escalation.

## Privilege Escalation

### Abusing cputils

As `www-data`, a `cputils` binary tied to the `zeus` context allowed copying
arbitrary files. It was used to copy `zeus`'s SSH private key.

```bash
cputils
# source: ./.ssh/id_rsa
# target: id_rsa
```

### Cracking the SSH key passphrase

The recovered key was passphrase-protected, so it was converted for John and
cracked with `rockyou.txt`.

```bash
ssh2john id_rsa > hash
john hash --wordlist=/usr/share/wordlists/rockyou.txt
```

The passphrase `snowflake` was recovered, enabling SSH access as `zeus`.

```bash
ssh -i id_rsa zeus@olympus.thm
```

This provided a stable shell. The user flag lives in `zeus`'s home directory.

### Hidden backdoor to root

As `zeus`, enumeration under `/var/www/html` revealed a randomly named directory
containing a password-protected PHP **backdoor**. The backdoor invoked
`/lib/defended/libc.so.99`, a non-standard binary running with elevated
privileges. Executing the binary indicated by the backdoor yielded a root
context, and the root flag lives at `/root/root.txt`.

## Vulnerability Analysis

**SQL injection** — the `cat_id` parameter in `category.php` allowed full
database enumeration and exfiltration of hashes and internal messages. Fix: use
parameterized queries / prepared statements and least-privilege DB accounts.

**Sensitive data exposure** — internal chat messages, filenames and password
hashes were readable via SQLi, mapping out the next stages of the attack. Fix:
avoid storing operational hints and secrets in queryable tables.

**Credential reuse** — a cracked database password was valid on the chat
application. Fix: enforce unique credentials per service and rotate on
disclosure.

**Insecure file upload** — the chat allowed uploading PHP files to a
web-interpreted directory, enabling RCE. Fix: validate file type/extension,
store uploads outside the web root, and disable execution in upload directories.

**Misuse of an auxiliary binary** — `cputils` could copy arbitrary files,
including an SSH private key. Fix: remove unnecessary helper binaries and apply
least privilege to file-access tooling.

**Hidden privileged artifact** — a password-protected backdoor and a privileged
binary in `/lib/defended/` provided the final escalation path. Fix: monitor for
unauthorized SUID/privileged binaries and unexpected files in system paths.

## Tools Used

- Nmap
- Feroxbuster
- sqlmap
- John the Ripper
- Netcat
- SSH

## Key Takeaways

- A single dynamic parameter like `cat_id` can compromise an entire application.
- Database dumps reveal internal application flows, not just credentials.
- Chat messages and filenames can hand over the exact exploitation path.
- An open SSH service is worth pivoting to for a more stable shell.
- Non-standard "helper" binaries and hidden artifacts often hide the privesc path.
