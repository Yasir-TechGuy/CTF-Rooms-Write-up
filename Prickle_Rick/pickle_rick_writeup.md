# CTF Write-up: Pickle Rick
**Platform:** TryHackMe  
**Difficulty:** Easy  
**Author:** Yasir  
**Status:** ✅ Completed  
**Room:** https://tryhackme.com/room/picklerick

> ⚠️ **Spoiler Warning:** This write-up contains hints and methodology. Try the room yourself before reading!

---

## Overview

Pickle Rick is a Rick and Morty themed CTF focused on web enumeration and command injection. The goal is to find 3 ingredients to turn Rick back from a pickle. A great beginner room reinforcing web enumeration fundamentals.

| Item | Details |
|------|---------|
| Target | Web application |
| Flags | 3 ingredients |
| Skills | Web enumeration, page source, robots.txt, command injection |
| Tools | Nmap, Gobuster, Browser |

---

## Step 1 — Nmap Scan

```bash
nmap -p- --min-rate 5000 -sV <TARGET_IP>
```

Found:
- Port 22 — SSH
- Port 80 — HTTP (Apache)

> 💡 **Tip:** Always start with a full port scan — missing a port means missing an attack surface.

---

## Step 2 — Web Enumeration

Visited the web page and checked page source:

```bash
# View source: Ctrl+U
```

Found a username hidden in an HTML comment.

Checked robots.txt:
```
http://<TARGET_IP>/robots.txt
```

Found a suspicious string — this turned out to be a password!

> 💡 **Tip:** Always check robots.txt and page source on every web challenge. Developers leave sensitive data in both all the time.

---

## Step 3 — Directory Enumeration

```bash
gobuster dir -u http://<TARGET_IP> -w /usr/share/wordlists/dirb/common.txt
```

Discovered a login page. Logged in with credentials found in the source and robots.txt.

---

## Step 4 — Command Injection

The web panel had a command execution field. Standard commands like `cat` were blocked but bypassed using:

```bash
less <filename>
# or
grep . <filename>
```

Used command injection to read ingredient files across the filesystem:

```bash
# Ingredient 1 — web directory
less Sup3rS3cretPickl3Ingred.txt

# Ingredient 2 — home directory
less /home/rick/"second ingredients"

# Ingredient 3 — root directory
sudo less /root/3rd.txt
```

> 💡 **Tip:** When `cat` is blocked, try `less`, `grep`, `tac`, `head`, `tail` or `strings` — many filters only block `cat` specifically.

---

## Flags (Ingredients)

| Ingredient | Location | Method |
|-----------|----------|--------|
| 🥒 1st | Web directory | Command injection — less |
| 🥒 2nd | /home/rick/ | Command injection — less |
| 🥒 3rd | /root/ | sudo less — no password required |

---

## What I Learned

- Always view page source — usernames and hints hide in HTML comments
- robots.txt can contain passwords and sensitive paths
- Gobuster finds hidden pages not visible from the main site
- Command filters are often incomplete — try alternative commands
- sudo -l (or just try sudo) — sometimes no password is required!
- Web command panels are a classic initial access vector

---

*Write-up by Yasir | TryHackMe Junior Penetration Tester Path*  
*Room: [Pickle Rick](https://tryhackme.com/room/picklerick)*
