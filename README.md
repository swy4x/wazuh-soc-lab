# Wazuh SOC Lab

A hands-on SOC lab built around Wazuh, Suricata, Zeek, controlled reconnaissance/authentication testing, FIM, malware-test artifacts, and YARA-based file hunting and automated detection.

## 🧰 Tools Corner

**Start here if you want to understand the lab's tools, what each one does, where it runs, and exactly how the tools connect to Wazuh.**

→ See [docs/tools-corner.md](docs/tools-corner.md)

The Tools Corner contains the A-to-Z tool map, architecture flows, ports, important paths, detection layers, Wazuh connections, YARA Active Response integration, and the practical learning order.

## Current stack

- Wazuh Manager 4.14.1
- Wazuh Indexer
- Wazuh Dashboard
- Wazuh Agent 4.14.5 on the Arch Linux host
- Suricata 8.x on the Arch Linux host
- Zeek on the Arch Linux host
- YARA 4.5.6
- Docker-based Wazuh deployment
- systemd journald collection for host authentication logs
- Arch Linux host with a Windows VM planned for later

## Architecture / data flow

### Network telemetry

```
Network traffic
      ├──────────────→ Suricata
      │                    ↓
      │             /var/log/suricata/eve.json
      │                    ↓
      │               Wazuh Manager
      │                    ↓
      │              Wazuh rules / alerts
      │                    ↓
      │               Wazuh Dashboard
      │
      └──────────────→ Zeek
                           ↓
                    Zeek logs / JSON
                           ↓
                     Wazuh Manager
                           ↓
                    Wazuh rules / alerts
                           ↓
                    Wazuh Dashboard
```

### Host authentication telemetry

```
sshd
  ↓
systemd journal
  ↓
Wazuh Agent
  ↓
Wazuh Manager
  ↓
Wazuh rules / alerts
  ↓
Wazuh Dashboard
```

### FIM → YARA → Wazuh workflow

```
File created/modified
  ↓
Wazuh FIM / syscheckd
  ↓
Rule 554 / Rule 550
  ↓
Wazuh Active Response
  ↓
Agent-side yara-scan
  ↓
YARA 4.5.6
  ↓
YARA_MATCH
  ↓
/var/ossec/logs/yara-results.log
  ↓
Wazuh Agent log collection
  ↓
Wazuh Manager Rule 100500
  ↓
Wazuh Dashboard Level 12 alert
```

## Labs completed

### Wazuh / endpoint

- systemd journald collection for SSH authentication telemetry
- SSH invalid-user authentication detection
- Wazuh SSH Rule 5710 observation
- SSH brute-force correlation using custom Wazuh Rule 100003
- Realtime Wazuh FIM for file creation, modification, and deletion
- EICAR test-file detection through Wazuh FIM
- EICAR SHA-256 IOC investigation and VirusTotal correlation

### Suricata

- Suricata → Wazuh alert ingestion
- DNS reconnaissance detection
- UDP scan / burst detection
- TCP SYN scan / port sweep detection
- Nmap-generated TCP SYN scan observed in Wazuh
- Custom Suricata SIDs for TCP, ICMP, SSH, SMB, RDP, HTTP, HTTPS, UDP, DNS, FIN, NULL, and Xmas traffic

### Zeek

- Zeek → Wazuh integration
- Synthetic SSL event detection
- Zeek DNS reconnaissance detection and correlation
- Zeek `id` field mapping conflict resolved by mapping the value to `data.zeek_id`

### YARA

- Exact string matching
- `any of them` / `all of them`
- Boolean AND / OR / NOT logic
- File-size conditions
- Compound conditions
- SHA-256 hash matching
- Hash-vs-content comparison using a modified EICAR artifact
- Regex matching
- Metadata
- Private helper rules
- Rule tags
- ELF module
- PE module and PE architecture testing
- PE + indicator logic
- Recursive EICAR hunting
- Automated YARA → Wazuh integration
- Wazuh Active Response invoking YARA
- Wazuh Rule 100500 generating a Level 12 YARA alert

## Repository structure

```
README.md
docs/
  tools-corner.md
  architecture.md
  yara-practical.md
  yara-wazuh-integration.md
network/
  zeek-wazuh.md
suricata/
  README.md
  local-rules.md
attacks/
  ssh-invalid-user.md
  ssh-brute-force.md
  nmap-tcp-syn-scan.md
  eicar-fim-detection.md
investigations/
  eicar-hash-investigation.md
```

## Evidence / screenshots

Screenshots are intentionally collected during each lab and added manually.

Use the screenshot markers inside the individual lab documents to keep evidence tied to the exact step being demonstrated.

## Repository goals

This repository documents the lab as it is actually built and tested. Detection rules, investigation notes, screenshots, and future Windows/Sysmon labs will be added incrementally.

> All testing is performed against systems and networks owned or controlled for this lab.
