# Wazuh SOC Lab

A hands-on SOC lab built around Wazuh, Suricata, and controlled reconnaissance testing.

## Current stack

- Wazuh Manager 4.14.1
- Wazuh Indexer
- Wazuh Dashboard
- Suricata 8.x on the Arch Linux host
- Docker-based Wazuh deployment
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

## Labs completed

- Suricata → Wazuh alert ingestion
- DNS reconnaissance detection
- UDP scan / burst detection
- TCP SYN scan / port sweep detection
- Nmap-generated TCP SYN scan observed in Wazuh

## Repository goals

This repository documents the lab as it is actually built and tested. Detection rules, investigation notes, screenshots, and future Windows/Sysmon labs will be added incrementally.

> All testing is performed against systems and networks owned or controlled for this lab.
