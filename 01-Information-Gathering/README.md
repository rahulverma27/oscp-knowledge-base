# Information Gathering

This section documents reconnaissance techniques and enumeration workflows used during penetration tests and OSCP-style labs.

The content here is written as **personal operational notes** while studying and practicing offensive security.

---

## Reference Playbooks

The following repositories influenced the structure of this section:

- https://github.com/Toecuta/OSCP
- https://github.com/floudeciel/OSCP-Human-Guide
- https://github.com/swisskyrepo/PayloadsAllTheThings
- https://github.com/carlospolop/PEASS-ng
- https://github.com/danielmiessler/SecLists

These projects provide extensive community knowledge for penetration testing and red team operations.

---

## Recon Workflow

Typical reconnaissance process:

1. Discover live hosts
2. Identify open ports
3. Detect running services
4. Enumerate services
5. Identify possible vulnerabilities
6. Prioritize attack surface

---

## Core Recon Categories

### Network Discovery
Host discovery and network mapping.

### Port Scanning
Identifying exposed services and open ports.

### Service Enumeration
Gathering detailed information about discovered services.

### DNS & Subdomain Enumeration
Identifying domains, records, and potential external attack surface.

### Web Technology Identification
Determining frameworks, servers, and technologies used by web applications.

---

## Tools Commonly Used

Network & discovery tools frequently used during reconnaissance:

- Nmap
- Amass
- theHarvester
- dnsrecon
- WhatWeb
- Wappalyzer

---

## Notes

Commands and detailed examples will be added here as they are used during labs and exercises.

The goal of this repository is to build a **personal penetration testing playbook** rather than replicate existing resources.
