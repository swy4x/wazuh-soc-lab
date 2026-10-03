# Wazuh SOC Lab Architecture

## 1. Central architecture

Wazuh is the central SOC platform.

```
                    Wazuh Dashboard
                           ↑
                           │
                    Wazuh Indexer
                           ↑
                           │
                    Wazuh Manager
                 ↙         ↑         ↖
                ↙          │          ↖
          Wazuh Agent   Zeek       Suricata
             ↑             │          │
       ┌─────┴─────┐       │          │
       │           │       │          │
    journald      FIM      └──────┬───┘
       ↑           │              │
      sshd         ↓          Network traffic
                 YARA
```

## 2. Wazuh components

### Wazuh Agent

Runs on the Arch Linux endpoint.

Responsibilities in this lab:

- collect host logs;
- collect FIM events;
- forward telemetry to the Manager;
- execute selected Active Response commands.

Agent:

- ID: `001`
- Name: `archlinux`
- Manager: `127.0.0.1`
- Version: `4.14.5`

### Wazuh Manager

Receives events, decodes them, applies rules/correlation, generates alerts, and controls Active Response.

### Wazuh Indexer

Stores alert/event data for search and investigation.

### Wazuh Dashboard

Provides the analyst interface for searching and investigating the stored events.

## 3. Host authentication pipeline

```
SSH authentication attempt
        ↓
      sshd
        ↓
systemd journald
        ↓
  Wazuh Agent
        ↓
 Wazuh Manager
        ↓
SSH rules / correlation
        ↓
Indexer / Dashboard
```

The agent collects the systemd journal using:

```xml
<localfile>
  <log_format>journald</log_format>
  <location>journald</location>
</localfile>
```

Important rules observed:

- 5710 — invalid-user SSH attempt
- 5760 — SSH authentication failure
- 5763 — built-in SSH brute-force correlation
- 100003 — custom SSH brute-force correlation

## 4. Suricata pipeline

```
Network traffic
      ↓
  Suricata
      ↓
Custom Suricata signature
      ↓
/var/log/suricata/eve.json
      ↓
 Wazuh ingestion
      ↓
Wazuh rule 86601
      ↓
Dashboard
```

In the tested TCP SYN scan, Suricata generated signature `1000001`. Wazuh then represented the ingested Suricata event with rule `86601`.

These are different detection layers.

## 5. Zeek pipeline

```
Network activity
      ↓
     Zeek
      ↓
Structured Zeek telemetry
      ↓
Ingestion pipeline
      ↓
     Wazuh
      ↓
Rules / correlation
      ↓
Dashboard
```

A field-mapping conflict with Zeek's native `id` field was resolved by mapping that value to:

```
data.zeek_id
```

The lab validated SSL telemetry and DNS reconnaissance/correlation.

## 6. FIM → YARA → Wazuh

```
File created / modified
        ↓
 Wazuh syscheckd
        ↓
   Rule 554 / 550
        ↓
 Active Response
        ↓
   yara-scan
        ↓
      YARA
        ↓
   YARA_MATCH
        ↓
/var/ossec/logs/yara-results.log
        ↓
 Wazuh Agent
        ↓
 Manager Rule 100500
        ↓
 Dashboard
```

The YARA integration was validated end-to-end with the harmless EICAR test artifact.

## 7. Key paths

| Purpose | Path |
|---|---|
| Suricata JSON | `/var/log/suricata/eve.json` |
| YARA rules | `/opt/yara-rules/wazuh-malware-lab.yar` |
| YARA result log | `/var/ossec/logs/yara-results.log` |
| YARA Active Response | `/var/ossec/active-response/bin/yara-scan` |
| Malware-test directory | `/opt/wazuh-malware-lab` |
| FIM database | `/var/ossec/queue/fim/db/fim.db` |

## 8. Design principle

Each tool has a defined role:

- **Wazuh** centralizes and correlates.
- **Suricata** detects network signatures.
- **Zeek** provides network telemetry.
- **FIM** detects endpoint file changes.
- **YARA** evaluates file content/properties.
- **VirusTotal** supports IOC investigation.

The value of the lab comes from connecting these layers into observable detection chains.
