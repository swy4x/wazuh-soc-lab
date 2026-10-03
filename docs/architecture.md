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
Provides the SOC analyst interface for threat hunting and alert investigation.

### YARA
YARA is used as a file-hunting engine and rule language. In this lab it runs on the Arch Linux endpoint and is invoked automatically by Wazuh Active Response for selected FIM events.

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

### FIM / YARA / Wazuh

```
File creation or modification
  ↓
Wazuh syscheckd / FIM
  ↓
Rule 554 / Rule 550
  ↓
Wazuh Active Response
  ↓
Agent-side yara-scan
  ↓
YARA
  ↓
YARA_MATCH
  ↓
/var/ossec/logs/yara-results.log
  ↓
Wazuh Agent
  ↓
Wazuh Manager Rule 100500
  ↓
Dashboard Level 12 alert
```

The YARA integration was validated end-to-end with the harmless EICAR test artifact.

## YARA integration details

- YARA version: **4.5.6**
- YARA rules: `/opt/yara-rules/wazuh-malware-lab.yar`
- Lab directory: `/opt/wazuh-malware-lab`
- Result log: `/var/ossec/logs/yara-results.log`
- Active Response command: `yara-scan`
- Triggering Wazuh rules: **554, 550**
- Final Wazuh rule: **100500**
- Final Wazuh alert level: **12**

The Active Response script reads one JSON line from stdin, extracts the FIM path, restricts scanning to the lab directory, runs YARA, and writes a normalized `YARA_MATCH` line to the result log.

## SOC architecture principle

Suricata and Zeek provide different network-visibility layers, while Wazuh provides centralized collection, rule processing, correlation, storage, and investigation. YARA provides file-content and file-property hunting and is now connected to Wazuh through FIM and Active Response.

See `docs/yara-wazuh-integration.md` for the complete implementation and validation details.
