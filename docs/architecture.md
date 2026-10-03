# Wazuh SOC Lab Architecture

> **The architecture explains how the individual tools become one SOC workflow.**

---

# 1.  Central Architecture

```
                         ┌─────────────────────┐
                         │  Wazuh Dashboard    │
                         │  Analyst / Search    │
                         └──────────▲──────────┘
                                    │
                         ┌──────────┴──────────┐
                         │   Wazuh Indexer     │
                         │   Storage / Search  │
                         └──────────▲──────────┘
                                    │
                         ┌──────────┴──────────┐
                         │   Wazuh Manager     │
                         │ Decode • Rules      │
                         │ Correlation • Alert │
                         │ Active Response     │
                         └─────┬────┬────┬──────┘
                               │    │    │
                     ┌─────────┘    │    └──────────┐
                     │              │               │
              ┌──────▼──────┐ ┌────▼────┐    ┌─────▼─────┐
              │ Wazuh Agent │ │  Zeek   │    │ Suricata  │
              └──────┬──────┘ └─────────┘    └───────────┘
                     │
              ┌──────┼──────────┐
              │      │          │
           journald  FIM       YARA
              │      │          │
             sshd   files   Active Response
```

The central design principle is simple:

> **Specialized tools generate or analyze telemetry; Wazuh connects the layers into a SOC workflow.**

---

# 2.  Wazuh Components

## Wazuh Agent

Runs on the Arch Linux endpoint.

Responsibilities:

- collect host logs;
- collect FIM events;
- forward telemetry;
- execute selected Active Response commands.

Identity:

| Field | Value |
|---|---|
| ID | 001 |
| Name | archlinux |
| Manager | 127.0.0.1 |
| Version | 4.14.5 |

---

## Wazuh Manager

The Manager is the central analysis layer.

It:

1. receives events;
2. decodes them;
3. applies rules;
4. correlates events;
5. generates alerts;
6. coordinates Active Response.

Container:

```
single-node-wazuh.manager-1
```

Version:

```
4.14.1
```

---

## Wazuh Indexer

Stores searchable alert/event data.

Port:

```
9200
```

---

## Wazuh Dashboard

The analyst interface.

Lab exposure:

```
443 → 5601
```

---

# 3.  Host Authentication Pipeline

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
     decoder + rules
            ↓
      correlation
            ↓
       Indexer
            ↓
       Dashboard
```

The Agent collects journald with:

```xml
<localfile>
  <log_format>journald</log_format>
  <location>journald</location>
</localfile>
```

### Important rules

| Rule | Meaning |
|---:|---|
| 5710 | Invalid SSH user |
| 5760 | SSH authentication failure |
| 5763 | Built-in SSH brute-force correlation |
| 100003 | Custom SSH brute-force correlation |
| 2502 | Repeated password failures |
| 5503 | PAM login failure |
| 5557 | Password check failure |

---

# 4.  SSH Correlation Architecture

The custom rule uses Rule 5760 as its input.

```
Failure #1 ─┐
Failure #2 ─┼── same source IP ── within 60 sec ──→ Rule 100003
Failure #3 ─┘
```

The important difference:

```
one failed login
       =
single event

repeated failures
       =
behavioral pattern
```

This is the first major example in the lab of Wazuh acting as a **correlation engine**, not merely a log viewer.

---

# 5.  Suricata Architecture

```
Network traffic
      ↓
   Suricata
      ↓
local signature
      ↓
/var/log/suricata/eve.json
      ↓
 Wazuh ingestion
      ↓
Wazuh Rule 86601
      ↓
 Dashboard
```

Suricata operates on:

```
wlp8s0
```

A controlled Nmap SYN scan generated:

```
Suricata SID 1000001
```

Wazuh then represented the ingested event under:

```
Rule 86601
```

### Important

These IDs belong to different systems.

```
1000001 → Suricata detection signature
86601   → Wazuh ingestion/detection rule
```

---

# 6.  Zeek Architecture

```
Network activity
      ↓
     Zeek
      ↓
structured telemetry
      ↓
ingestion / Filebeat pipeline
      ↓
    Wazuh
      ↓
rules / correlation
      ↓
 Dashboard
```

The lab validated:

- synthetic SSL telemetry;
- DNS reconnaissance telemetry;
- DNS reconnaissance correlation.

Custom rules:

| Rule | Level | Purpose |
|---|---:|---|
| 100102 | 3 | Synthetic SSL |
| 100304 | 7 | DNS reconnaissance |
| 100305 | 10 | DNS reconnaissance correlation |

---

# 7.  Zeek Mapping Problem

Zeek uses a native field:

```
id
```

The ingestion mapping encountered a conflict around this field.

The working solution was:

```
data.zeek_id
```

### Why this matters

Security integrations do not fail only because of detection logic.

They can also fail because:

- fields have conflicting types;
- schemas differ;
- one system reserves a field name;
- ingestion pipelines transform data incorrectly.

This was a real debugging step in the lab.

---

# 8.  FIM Architecture

FIM monitors the endpoint filesystem.

Realtime directory:

```
/opt/wazuh-malware-lab
```

Pipeline:

```
file creation / modification / deletion
                    ↓
                 inotify
                    ↓
            wazuh-syscheckd
                    ↓
               Wazuh Agent
                    ↓
              Wazuh Manager
                    ↓
                FIM rules
                    ↓
                  alert
```

Rules:

| Rule | Meaning |
|---:|---|
| 554 | File added |
| 550 | File modified / checksum changed |
| 553 | File deleted |

FIM database:

```
/var/ossec/queue/fim/db/fim.db
```

The lab also verified that the syscheck process was using Linux inotify.

---

# 9.  EICAR Investigation Architecture

```
EICAR file
   ↓
FIM Rule 554
   ↓
file metadata + hashes
   ↓
SHA-256 verification
   ↓
VirusTotal investigation
   ↓
analyst interpretation
```

The EICAR file was never executed.

This workflow demonstrates an important SOC pattern:

> **Detection is not the same thing as investigation.**

FIM tells us that something happened.

Hash/IOC investigation helps determine what that thing is.

---

# 10.  YARA Architecture

YARA is the file-analysis layer.

```
File
 ↓
YARA rule
 ↓
YARA engine
 ↓
match / no match
```

The practical YARA phase tested:

- strings;
- Boolean logic;
- file size;
- hashes;
- regex;
- metadata;
- helper rules;
- tags;
- ELF;
- PE;
- recursive scanning.

The formal YARA-language course is kept separate.

---

# 11.  FIM → YARA → Wazuh

The completed end-to-end chain is:

```
File created / modified
          ↓
       Wazuh FIM
          ↓
      Rule 554/550
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
       Indexer
          ↓
      Dashboard
```

This is one of the most important completed workflows in the repository.

---

# 12.  Why the YARA Integration Has Two Detection Steps

The architecture deliberately separates:

### FIM

```
"What changed?"
```

### YARA

```
"Does this file match my detection logic?"
```

### Wazuh

```
"How should the SOC correlate, alert, store and investigate the result?"
```

This separation is more useful than treating everything as one giant scanner.

---

# 13.  YARA Active Response

Agent-side files:

```
/opt/yara-rules/wazuh-malware-lab.yar
/var/ossec/active-response/bin/yara-scan
/var/ossec/logs/yara-results.log
```

The scanner is restricted to:

```
/opt/wazuh-malware-lab/*
```

The Manager rule:

```
100500
```

converts the normalized:

```
YARA_MATCH
```

event into a Wazuh alert.

---

# 14.  Important Integration Debugging

The first scanner read stdin using:

```bash
INPUT="$(cat)"
```

This waited for EOF and left the Active Response process hanging.

The process was observed waiting on a pipe.

The working solution:

```bash
IFS= read -r INPUT
```

This reads one event and exits.

Another configuration error occurred when an Active Response block was initially placed after:

```xml
</ossec_config>
```

That produced an invalid-root configuration error.

The bad block was removed and the configuration was placed inside the valid Wazuh configuration structure.

Manager validation then succeeded.

These debugging steps are retained because they are part of the real engineering process.

---

# 15.  Key Paths

| Purpose | Path |
|---|---|
| Wazuh Docker | `~/wazuh-docker/single-node` |
| Malware-test directory | `/opt/wazuh-malware-lab` |
| YARA rule | `/opt/yara-rules/wazuh-malware-lab.yar` |
| Active Response | `/var/ossec/active-response/bin/yara-scan` |
| YARA result log | `/var/ossec/logs/yara-results.log` |
| FIM DB | `/var/ossec/queue/fim/db/fim.db` |
| Suricata events | `/var/log/suricata/eve.json` |

---

# 16.  Key Ports

```
1514-1515/tcp → Agent communication
514/udp        → Event input
55000/tcp      → Wazuh API
9200           → Indexer
443 → 5601     → Dashboard
```

---

# 17.  Complete Detection Model

```
                     SECURITY ACTIVITY
                             ↓
                 ┌───────────┴───────────┐
                 │                       │
              Endpoint                 Network
                 │                       │
       ┌─────────┴─────────┐       ┌─────┴─────┐
       │                   │       │           │
    journald              FIM   Suricata     Zeek
       │                   │
      sshd               files
                           │
                           ↓
                         YARA
                           │
                           └───────┐
                                   │
                              Wazuh Manager
                                   │
                            rules/correlation
                                   │
                                 alerts
                                   │
                              Indexer
                                   │
                              Dashboard
                                   │
                             investigation
```

---

# 18.  Architecture Principle

The lab is intentionally layered.

### Source

Produces evidence.

### Sensor / analyzer

Interprets the evidence.

### Wazuh

Centralizes and correlates it.

### Dashboard

Lets the analyst investigate it.

That gives the SOC analyst a repeatable mental model:

```
Where did the event originate?
        ↓
What telemetry was produced?
        ↓
Which tool detected/interpreted it?
        ↓
Which Wazuh rule fired?
        ↓
What evidence do I have?
        ↓
What should I investigate next?
```


# 19. Custom Detection Rules

The lab uses custom detection logic at several layers. The identifiers belong to different engines and must not be mixed.

## Wazuh Rule 100003

\`\`\`xml
<rule id="100003" level="10" frequency="3" timeframe="60" ignore="60">
  <if_matched_sid>5760</if_matched_sid>
  <same_source_ip/>
  <description>LAB: SSH Brute Force - 3 authentication failures from the same source IP within 60 seconds</description>
  <mitre><id>T1110</id></mitre>
  <group>authentication_failed,brute_force,ssh,lab,mitre_t1110,</group>
</rule>
\`\`\`

The rule consumes built-in Rule 5760 events and adds source-IP, frequency, timeframe, and suppression logic.

## Wazuh Rule 100500

\`\`\`xml
<rule id="100500" level="12">
  <match>YARA_MATCH</match>
  <description>YARA detected a malware-test indicator in a FIM-monitored file</description>
  <group>yara,malware_detection,file_integrity,lab,</group>
</rule>
\`\`\`

This is the final Wazuh detection layer in the FIM-to-YARA workflow.

## YARA integration rule

\`\`\`yara
rule Wazuh_EICAR_Test
{
    meta:
        author = "Swayam"
        description = "Detects the harmless EICAR antivirus test string"
        severity = "high"

    strings:
        $eicar = "EICAR-STANDARD-ANTIVIRUS-TEST-FILE"

    condition:
        $eicar
}
\`\`\`

## Custom network identifiers

| Engine | Identifier | Purpose |
|---|---:|---|
| Suricata | SID 1000001-1000012 | Controlled network signatures |
| Zeek/Wazuh | 100102 | Synthetic SSL |
| Zeek/Wazuh | 100304 | DNS reconnaissance |
| Zeek/Wazuh | 100305 | DNS reconnaissance correlation |

The repository records verified identifiers and behavior where original source bodies are available. It does not reconstruct unverified signature syntax.