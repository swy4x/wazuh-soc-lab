# Wazuh SOC Lab Architecture

## Components

### Wazuh Manager
Receives security events, decodes data, applies Wazuh rules, and generates alerts.

### Suricata
Acts as the network IDS. It inspects traffic and writes JSON events to:

`/var/log/suricata/eve.json`

### Wazuh Indexer
Stores Wazuh events so they can be searched and investigated.

### Wazuh Dashboard
Provides the SOC analyst interface for Threat Hunting and alert investigation.

## Practical pipeline

A tested network-scan event followed this path:

```
Nmap
  ↓
TCP SYN traffic
  ↓
Suricata
  ↓
Custom Suricata detection
  ↓
eve.json
  ↓
Wazuh Manager
  ↓
Wazuh rule 86601
  ↓
Indexer
  ↓
Dashboard
```

The Suricata detection and Wazuh ingestion rule are separate layers. In the tested event, Suricata used signature ID `1000001`, while Wazuh used rule `86601`.
