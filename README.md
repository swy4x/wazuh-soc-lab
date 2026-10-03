# Wazuh SOC Lab

A hands-on SOC lab built around Wazuh, Suricata, Zeek, controlled reconnaissance/authentication testing, FIM, malware-test artifacts, and YARA-based file hunting.

## Current stack

- Wazuh Manager 4.14.1
- Wazuh Indexer
- Wazuh Dashboard
- Wazuh Agent 4.14.5 on the Arch Linux host
- Suricata 8.x on the Arch Linux host
- Zeek on the Arch Linux host
- YARA 4.5.6 for standalone file hunting
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

### File / malware-test workflow

```
File appears
  ↓
Wazuh FIM
  ↓
Hash / metadata
  ↓
IOC investigation (VirusTotal)
  ↓
YARA file hunting
```

## Labs completed

- Suricata → Wazuh alert ingestion
- DNS reconnaissance detection
- UDP scan / burst detection
- TCP SYN scan / port sweep detection
- Nmap-generated TCP SYN scan observed in Wazuh
- Zeek → Wazuh integration
- Zeek synthetic SSL event detection
- Zeek DNS reconnaissance detection and correlation
- SSH invalid-user authentication detection through journald
- Wazuh SSH rule 5710 observed on the Arch Linux endpoint
- SSH brute-force correlation using custom Wazuh rule 100003
- Real-time Wazuh FIM: file creation, modification, and deletion
- EICAR test-file detection through Wazuh FIM
- EICAR SHA-256 IOC investigation and VirusTotal correlation
- Standalone YARA practicals: strings, regex, logic, filesize, hashes, metadata, tags, private helper rules, ELF and PE modules
- YARA hash-vs-string comparison using a harmless modified EICAR test artifact

## Evidence / screenshots

Screenshots are intentionally collected during each lab and added manually.

Use the screenshot markers inside the individual lab documents to keep evidence tied to the exact step being demonstrated.

## Repository goals

This repository documents the lab as it is actually built and tested. Detection rules, investigation notes, screenshots, and future Windows/Sysmon labs will be added incrementally.

> All testing is performed against systems and networks owned or controlled for this lab.
