# Wazuh SOC Lab Architecture

## Components

### Wazuh Manager
Receives security events, decodes data, applies Wazuh rules, and generates alerts.

### Suricata
Acts as the network IDS. It inspects traffic and writes JSON events to:

`/var/log/suricata/eve.json`

### Zeek
Acts as a network security monitor. It produces structured network telemetry such as connection, DNS, HTTP, SSL/TLS, and other protocol logs. In this lab, Zeek telemetry is forwarded to Wazuh for centralized detection and investigation.

### Wazuh Indexer
Stores Wazuh events so they can be searched and investigated.

### Wazuh Dashboard
Provides the SOC analyst interface for Threat Hunting and alert investigation.

### YARA
Runs as a standalone file-hunting engine in the current lab. YARA rules inspect file content, properties, hashes, and executable structures. YARA is not yet directly integrated into the Wazuh alert pipeline.

## Practical pipelines

### Suricata

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

### Zeek

```
Controlled network activity
  ↓
Zeek
  ↓
Zeek structured telemetry
  ↓
Filebeat / ingestion pipeline
  ↓
Wazuh Manager
  ↓
Wazuh rules
  ↓
Wazuh Dashboard
```

The lab validated Zeek telemetry reaching Wazuh, including a synthetic SSL event and DNS reconnaissance events. A field-mapping conflict involving Zeek's `id` field was resolved by mapping the value to `data.zeek_id` before ingestion.

### Host authentication

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

### FIM / malware-test workflow

```
File creation or modification
  ↓
Wazuh syscheckd / FIM
  ↓
Agent
  ↓
Manager
  ↓
FIM rule
  ↓
Dashboard
```

For EICAR, the resulting SHA-256 was independently verified and investigated as a known harmless antivirus test artifact.

## SOC architecture principle

Suricata and Zeek provide different network-visibility layers, while Wazuh provides centralized collection, rule processing, correlation, storage, and investigation. YARA currently remains a separate file-hunting tool in the lab; a future exercise can integrate YARA results into Wazuh.
