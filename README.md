# Wazuh SOC Lab

A hands-on SOC lab built around Wazuh, Suricata, and controlled reconnaissance/authentication testing.

## Current stack

- Wazuh Manager 4.14.1
- Wazuh Indexer
- Wazuh Dashboard
- Wazuh Agent 4.14.5 on the Arch Linux host
- Suricata 8.x on the Arch Linux host
- Docker-based Wazuh deployment
- systemd journald collection for host authentication logs
- Arch Linux host with a Windows VM planned for later

## Data flow

```
Network traffic
      ↓
   Suricata
      ↓
 /var/log/suricata/eve.json
      ↓
 Wazuh Manager
      ↓
 Wazuh rules / alerts
      ↓
 Wazuh Indexer
      ↓
 Wazuh Dashboard
```

Host authentication flow:

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

## Labs completed

- Suricata → Wazuh alert ingestion
- DNS reconnaissance detection
- UDP scan / burst detection
- TCP SYN scan / port sweep detection
- Nmap-generated TCP SYN scan observed in Wazuh
- SSH invalid-user authentication detection through journald
- Wazuh SSH rule 5710 observed on the Arch Linux endpoint
- SSH brute-force correlation using custom Wazuh rule 100003

## Repository goals

This repository documents the lab as it is actually built and tested. Detection rules, investigation notes, screenshots, and future Windows/Sysmon labs will be added incrementally.

> All testing is performed against systems and networks owned or controlled for this lab.
