# Information Gathering

*The foundation of every successful penetration test.*

Information gathering (reconnaissance) is the process of **systematically discovering assets, services, and potential attack surfaces** within a target environment.

A disciplined recon methodology dramatically reduces exploitation time and reveals attack paths that would otherwise remain hidden.

This section documents my **personal reconnaissance workflow**, refined through penetration-testing labs and methodologies inspired by the community resources such as PayloadsAllTheThings, OSCP-Human-Guide, and SecLists.

---

# Recon Philosophy

The objective of reconnaissance is not to run tools.

The objective is to **build an attack map**.

A good recon workflow answers five questions:

1. What hosts exist?
2. What services run on them?
3. What technologies are used?
4. What trust relationships exist?
5. Which attack vectors are most promising?

Recon follows a **funnel model**:

```
Broad Discovery
      ↓
Host Enumeration
      ↓
Service Enumeration
      ↓
Technology Identification
      ↓
Attack Surface Mapping
```

---

# Recon Workflow

## Phase 1 — Host Discovery

Determine which systems are alive in the target network.

### Ping Sweep

```bash
nmap -sn 192.168.1.0/24
```

### TCP Ping

```bash
nmap -sn -PS21,22,80,445 192.168.1.0/24
```

### Fast Discovery with fping

```bash
fping -a -g 192.168.1.0/24 2>/dev/null
```

### ARP Discovery

```bash
netdiscover -r 192.168.1.0/24
```

ARP scanning is often the **most reliable technique inside local networks**.

---

# Phase 2 — Port Scanning

Once live hosts are identified, enumerate open ports.

## Quick Scan

```bash
nmap -T4 -F 10.10.10.10
```

## Full Port Scan

```bash
nmap -sS -p- --min-rate 1000 10.10.10.10
```

## Version Detection

```bash
nmap -sV -sC -p <open_ports> 10.10.10.10
```

## Complete Enumeration

```bash
nmap -sS -sV -sC -O -p- -T4 10.10.10.10
```

Output should always be saved for analysis.

```
-oA scan_results
```

---

# High Speed Scanning

For large environments, use **Masscan**.

```bash
masscan -p1-65535 10.10.10.10 --rate=10000
```

Masscan discovers ports extremely quickly, but service enumeration must still be performed with Nmap.

---

# Phase 3 — Service Enumeration

Once open ports are discovered, enumerate each service individually.

---

# HTTP Enumeration

## Technology Identification

```bash
whatweb http://10.10.10.10
```

## Directory Discovery

```bash
gobuster dir -u http://10.10.10.10 \
-w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

## Web Server Vulnerability Scan

```bash
nikto -h http://10.10.10.10
```

## Manual Inspection

```bash
curl -I http://10.10.10.10
```

Manual analysis often reveals headers, authentication methods, or misconfigurations.

---

# SMB Enumeration

SMB frequently reveals **shares, users, and domain information**.

### List Shares

```bash
smbclient -L //10.10.10.10 -N
```

### SMB Share Enumeration

```bash
smbmap -H 10.10.10.10
```

### Domain Enumeration

```bash
enum4linux-ng -A 10.10.10.10
```

### RID Brute Force

```bash
crackmapexec smb 10.10.10.10 --rid-brute
```

This technique often reveals domain users.

---

# DNS Enumeration

DNS can expose subdomains, infrastructure, and internal naming schemes.

### Zone Transfer Attempt

```bash
dig axfr @10.10.10.10 domain.local
```

### Subdomain Enumeration

```bash
gobuster dns -d domain.local \
-w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-5000.txt
```

### Automated Enumeration

```bash
dnsrecon -d domain.local -a
```

---

# Passive Reconnaissance

Passive recon gathers intelligence **without interacting directly with the target infrastructure**.

## Email & Domain Intelligence

```bash
theHarvester -d domain.local -b google,bing
```

## OSINT Enumeration

```bash
amass enum -passive -d domain.local
```

## Internet Exposure Search

```bash
shodan search hostname:domain.local
```

Passive recon often reveals:

* exposed services
* forgotten subdomains
* leaked credentials
* shadow infrastructure

---

# Service Specific Enumeration

Once a service is identified, use targeted scripts.

## FTP

```bash
nmap -p21 --script ftp* 10.10.10.10
```

## SSH

```bash
nmap -p22 --script ssh* 10.10.10.10
```

## MySQL

```bash
nmap -p3306 --script mysql* 10.10.10.10
```

Service-specific scripts often reveal **misconfigurations and default credentials**.

---

# Recon Automation

A simple script can accelerate initial enumeration.

```bash
#!/bin/bash

TARGET=$1

echo "[*] Host discovery"
nmap -sn $TARGET/24 -oG hosts.txt

echo "[*] Port scanning"
nmap -sS -T4 -F $TARGET -oA quick_scan
```

Automation ensures consistency during time-limited assessments.

---

# Recon Output Organization

A structured output directory prevents confusion during large engagements.

```
recon
 ├── hosts
 ├── ports
 ├── services
 ├── web
 ├── smb
 └── notes
```

Good organization dramatically improves reporting and exploitation efficiency.

---

# Practical Recon Rules

### 1. Never trust a single scan

Always verify results with multiple tools.

### 2. Always enumerate deeper

The most valuable vulnerabilities are usually hidden.

### 3. Document everything

Every finding can become an attack vector later.

### 4. Correlate results

Attack paths emerge from relationships between systems.

---

# Personal Recon Checklist

Before moving to exploitation, confirm:

* Live hosts identified
* Full TCP scan completed
* UDP scan attempted
* Web directories enumerated
* SMB shares checked
* DNS investigated

If any step is missing, recon is incomplete.

---

# References

Key community resources used while building this playbook:

* PayloadsAllTheThings
* SecLists
* OSCP-Human-Guide

---

# Status

OSCP Preparation: In Progress
Recon Methodology: Continuously Updated

---

