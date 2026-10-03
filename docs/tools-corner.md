# 🧰 Tools Corner — Wazuh SOC Lab

This page is the practical **tool map** for the lab.

It explains:

- what each tool does;
- why we use it;
- where it runs;
- what it sends/receives;
- how it connects to Wazuh;
- which part of the SOC workflow it belongs to;
- the important paths, ports, rules, and commands used in the lab.

The goal is to understand the lab as a real SOC environment rather than as a collection of unrelated commands.

---

# 1. The Big Picture

The lab is built around **Wazuh as the central SOC platform**.

```
                         ┌───────────────┐
                         │   Wazuh       │
                         │   Dashboard   │
                         └───────▲───────┘
                                 │
                         Alerts / Investigation
                                 │
                         ┌───────┴───────┐
                         │ Wazuh Manager  │
                         │ Rules / Correlation
                         └───▲───────┬───┘
                             │       │
                    Telemetry│       │Active Response
                             │       ▼
              ┌──────────────┘   ┌──────────────┐
              │                  │ Wazuh Agent  │
              │                  └──────▲───────┘
              │                         │
       ┌──────┴──────┐        ┌─────────┴────────┐
       │  Suricata   │        │  FIM / syscheckd │
       │  Network IDS│        │  Endpoint files  │
       └──────┬──────┘        └─────────┬────────┘
              │                         │
       Network traffic             File events
                                        │
                                  ┌─────▼─────┐
                                  │   YARA    │
                                  │ File hunt │
                                  └───────────┘

       ┌──────────────┐
       │     Zeek     │
       │ Network      │
       │ telemetry    │
       └──────┬───────┘
              │
              └──────────────→ Wazuh ingestion

       ┌──────────────┐
       │    sshd      │
       │ systemd      │
       │ journald     │
       └──────┬───────┘
              │
              └──────────────→ Wazuh Agent
```

---

# 2. Core Tool Stack

| Tool | Role | Where | Connection to Wazuh |
|---|---|---|---|
| **Wazuh Manager** | Central analysis, rules, alerts, correlation | Docker | Core |
| **Wazuh Agent** | Endpoint telemetry + response | Arch Linux host | Direct |
| **Wazuh Dashboard** | SOC investigation UI | Docker | Reads Wazuh alerts |
| **Wazuh Indexer** | Stores/searches events | Docker | Backend |
| **Suricata** | Network IDS | Arch host | Logs → Wazuh |
| **Zeek** | Network security monitoring | Arch host | Telemetry → Wazuh |
| **YARA** | File/content hunting | Arch host | Active Response → Wazuh |
| **Docker** | Runs Wazuh stack | Arch host | Hosts Wazuh components |
| **systemd journald** | Host log source | Arch host | Agent collects journal |
| **sshd** | Authentication event source | Arch host | journal → Agent |
| **Nmap** | Controlled network testing | Test device | Traffic → Suricata |
| **VirusTotal** | IOC/hash investigation | External service | Analyst investigation |
| **jq** | JSON parsing in scripts | Arch host | Used by YARA response |
| **SQLite** | FIM database inspection | Arch host | Used for learning/investigation |
| **Termux** | Controlled test client | Test device | Generated test traffic |

---

# 3. Wazuh — The Center

## Wazuh Manager

The Manager is the central brain of the lab.

It:

- receives events;
- decodes incoming data;
- evaluates Wazuh rules;
- generates alerts;
- performs correlation;
- triggers Active Response;
- sends searchable alert data toward the Indexer/Dashboard.

### Important Manager rules used in the lab

- **5710** — SSH invalid-user authentication event
- **5760** — SSH authentication failure correlation source
- **5763** — built-in SSH brute-force correlation
- **100003** — custom SSH brute-force lab rule
- **554** — FIM file added
- **550** — FIM file modified/checksum changed
- **553** — FIM file deleted
- **86601** — Suricata event ingestion observed in the lab
- **100500** — YARA detection converted into a Wazuh Level 12 alert

---

# 4. Wazuh Agent

The Agent runs on the Arch Linux endpoint.

Its major jobs in this lab are:

```
Host logs
   ↓
Agent
   ↓
Manager

File changes
   ↓
FIM/syscheckd
   ↓
Agent
   ↓
Manager

FIM event
   ↓
Active Response
   ↓
YARA
```

### Agent

- ID: `001`
- Name: `archlinux`
- Manager address: `127.0.0.1`
- Agent package: `4.14.5-1`

The Agent configuration was repeatedly validated with:

```bash
sudo /var/ossec/bin/wazuh-agentd -t
```

---

# 5. Wazuh Dashboard

The Dashboard is the analyst interface.

We use it to:

- search alerts;
- inspect rule IDs;
- inspect rule levels;
- inspect agent information;
- inspect source IPs;
- inspect file paths;
- inspect hashes;
- investigate YARA results;
- verify that the entire detection pipeline reached the SOC interface.

The final YARA integration was confirmed in the Dashboard with:

```text
Rule: 100500
Level: 12
Agent: 001 / archlinux
Location: /var/ossec/logs/yara-results.log

YARA_MATCH rule=Wazuh_EICAR_Test path=/opt/wazuh-malware-lab/eicar.com
```

---

# 6. Wazuh Indexer

The Indexer stores the Wazuh alert/event data used for investigation.

The Docker deployment exposes the Indexer on:

```text
9200
```

The Dashboard uses the stored data for searching and investigation.

---

# 7. Docker

The Wazuh deployment is Docker-based.

Main services:

```text
single-node-wazuh.manager-1
single-node-wazuh.dashboard-1
single-node-wazuh.indexer-1
```

Important exposed ports:

```text
1514-1515/tcp  → Wazuh agent communication
514/udp         → Wazuh event input
55000/tcp       → Wazuh API
9200            → Wazuh Indexer
443 → 5601      → Wazuh Dashboard
```

Docker keeps the Wazuh Manager, Indexer, and Dashboard separated from the Arch host while the Wazuh Agent and security tools run on the host.

---

# 8. systemd journald

Arch Linux uses systemd journald for system logs.

Instead of making the Manager directly read the host journal, the **Wazuh Agent** collects it.

The Agent uses:

```xml
<localfile>
  <log_format>journald</log_format>
  <location>journald</location>
</localfile>
```

This gives the SOC visibility into host authentication events.

---

# 9. SSH / sshd

OpenSSH/sshd provides authentication events.

Example workflow:

```
SSH login attempt
       ↓
sshd
       ↓
systemd journal
       ↓
Wazuh Agent
       ↓
Wazuh Manager
       ↓
SSH detection rules
       ↓
Dashboard
```

We tested:

- invalid users;
- authentication failures;
- repeated failures from the same source;
- custom brute-force correlation.

Custom lab rule:

```text
Rule ID: 100003
Level: 10
Frequency: 3
Timeframe: 60 seconds
Ignore: 60 seconds
```

---

# 10. Suricata

Suricata is the lab's **network IDS**.

It inspects network traffic and generates structured events.

### Interface

```text
wlp8s0
```

### Main output

```text
/var/log/suricata/eve.json
```

### Data flow

```
Network traffic
      ↓
Suricata
      ↓
Custom/local detection rules
      ↓
eve.json
      ↓
Wazuh
      ↓
Dashboard
```

### Custom Suricata SIDs

| SID | Test |
|---:|---|
| 1000001 | TCP SYN |
| 1000002 | ICMP |
| 1000003 | SSH burst |
| 1000004 | SMB |
| 1000005 | RDP |
| 1000006 | HTTP |
| 1000007 | HTTPS |
| 1000008 | UDP |
| 1000009 | DNS burst |
| 1000010 | FIN |
| 1000011 | NULL |
| 1000012 | Xmas |

The Nmap TCP SYN test generated Suricata SID **1000001**, which was then visible through Wazuh ingestion.

---

# 11. Nmap

Nmap is used only for **controlled reconnaissance testing** in the lab.

Its purpose is not to be the detection system.

It generates network behavior.

```
Nmap
  ↓
TCP SYN traffic
  ↓
Suricata
  ↓
Detection
  ↓
Wazuh
  ↓
Dashboard
```

This gives the SOC lab a realistic way to generate reconnaissance telemetry.

---

# 12. Zeek

Zeek is the lab's **network security monitoring / network telemetry** layer.

It provides structured protocol-level information such as:

- connections;
- DNS;
- HTTP;
- SSL/TLS;
- file/network metadata.

### Data flow

```
Network activity
      ↓
Zeek
      ↓
Structured Zeek logs
      ↓
Filebeat / ingestion pipeline
      ↓
Wazuh
      ↓
Rules
      ↓
Dashboard
```

### Important integration fix

Zeek's `id` field conflicted with the Wazuh ingestion mapping.

We solved this by mapping the Zeek value to:

```text
data.zeek_id
```

instead of using the conflicting `id` field directly.

---

# 13. YARA

YARA is both:

1. a **rule language**;
2. a **file-hunting/detection engine**.

Version:

```text
YARA 4.5.6
```

We first learned YARA independently and tested:

- strings;
- `any of them`;
- `all of them`;
- AND / OR / NOT;
- filesize;
- compound conditions;
- SHA-256 hashes;
- regex;
- metadata;
- private helper rules;
- tags;
- ELF module;
- PE module;
- PE architecture;
- PE + indicator logic;
- recursive hunting.

Then we connected YARA to Wazuh.

---

# 14. How YARA Connects to Wazuh

This is one of the most important parts of the lab.

### Step 1 — Wazuh FIM watches the directory

```text
/opt/wazuh-malware-lab
```

with realtime monitoring.

### Step 2 — File event occurs

A file is created or modified.

### Step 3 — Wazuh generates a FIM event

Relevant rules:

```text
554 → file added
550 → file modified
```

### Step 4 — Active Response starts

Wazuh launches:

```text
yara-scan
```

for Rule 554 or Rule 550.

### Step 5 — yara-scan extracts the file path

The script receives a JSON event through stdin and extracts the FIM path.

It also restricts scanning to:

```text
/opt/wazuh-malware-lab/*
```

### Step 6 — YARA scans the file

YARA evaluates:

```text
/opt/yara-rules/wazuh-malware-lab.yar
```

### Step 7 — Result is logged

A match produces:

```text
YARA_MATCH rule=Wazuh_EICAR_Test path=/opt/wazuh-malware-lab/eicar.com
```

in:

```text
/var/ossec/logs/yara-results.log
```

### Step 8 — Wazuh collects that log

The Agent reads the result log as a syslog-formatted local file.

### Step 9 — Wazuh Rule 100500 fires

```xml
<rule id="100500" level="12">
  <match>YARA_MATCH</match>
  <description>YARA detected a malware-test indicator in a FIM-monitored file</description>
</rule>
```

### Step 10 — Dashboard shows the alert

Final pipeline:

```
File
 ↓
FIM
 ↓
554 / 550
 ↓
Active Response
 ↓
yara-scan
 ↓
YARA
 ↓
YARA_MATCH
 ↓
yara-results.log
 ↓
Agent
 ↓
Manager
 ↓
100500 / Level 12
 ↓
Dashboard
```

---

# 15. The YARA Active Response Script

Important files:

```text
/var/ossec/active-response/bin/yara-scan
/opt/yara-rules/wazuh-malware-lab.yar
/var/ossec/logs/yara-results.log
```

The script:

1. reads one JSON line;
2. checks for the `add` command;
3. extracts the FIM path;
4. validates the path;
5. runs YARA;
6. extracts the match;
7. writes `YARA_MATCH`;
8. exits.

### Important debugging lesson

The first implementation used:

```bash
INPUT="$(cat)"
```

This waited for EOF and caused the Active Response process to hang.

The working implementation uses:

```bash
IFS= read -r INPUT
```

This reads the single event line and lets the script finish.

---

# 16. FIM / syscheckd

Wazuh FIM is the endpoint file-integrity layer.

It detects:

- file creation;
- modification;
- deletion;
- metadata/hash changes.

We configured realtime monitoring for:

```text
/opt/wazuh-malware-lab
```

The FIM database is:

```text
/var/ossec/queue/fim/db/fim.db
```

SQLite was used to inspect the database structure while learning how FIM stores file state.

---

# 17. EICAR

EICAR is **not real malware**.

It is a standardized harmless antivirus test artifact used to validate security-product detection.

We used:

```text
/opt/wazuh-malware-lab/eicar.com
```

It was **never executed**.

Wazuh FIM detected its creation and recorded cryptographic hashes.

The SHA-256 was independently verified and investigated.

---

# 18. VirusTotal

VirusTotal was used as an **IOC investigation source**, not as part of the automatic Wazuh detection pipeline.

Workflow:

```
Wazuh FIM
   ↓
SHA-256
   ↓
VirusTotal lookup
   ↓
Vendor detections / labels
   ↓
Analyst interpretation
```

The EICAR SHA-256 produced a strong detection consensus, but the investigation correctly identified the artifact as the known harmless EICAR test file.

---

# 19. jq

`jq` is used by the YARA Active Response script.

Its job is to extract values from the JSON event supplied by Wazuh.

For example, the script uses JSON parsing to obtain the FIM file path.

Conceptually:

```
Wazuh JSON
    ↓
jq
    ↓
file path
    ↓
YARA
```

---

# 20. SQLite

SQLite was used during FIM investigation to inspect:

```text
/var/ossec/queue/fim/db/fim.db
```

This helped demonstrate that FIM maintains endpoint file state rather than simply printing a temporary message.

---

# 21. Termux

Termux was used as a controlled test client for network activity.

Example:

```
Termux / test device
        ↓
Nmap
        ↓
TCP SYN traffic
        ↓
Suricata
        ↓
Wazuh
```

It provided a separate source for generating network-test traffic against the lab.

---

# 22. Ports and Interfaces Cheat Sheet

| Component | Port / Interface |
|---|---|
| Wazuh Agent communication | 1514-1515/tcp |
| Wazuh event input | 514/udp |
| Wazuh API | 55000/tcp |
| Wazuh Indexer | 9200 |
| Wazuh Dashboard | 443 → 5601 |
| Suricata capture interface | `wlp8s0` |
| Wazuh Agent Manager address | `127.0.0.1` |

---

# 23. Important Lab Paths

| Purpose | Path |
|---|---|
| Wazuh Docker stack | `~/wazuh-docker/single-node` |
| Malware-test lab | `/opt/wazuh-malware-lab` |
| YARA rules | `/opt/yara-rules` |
| YARA integration rule | `/opt/yara-rules/wazuh-malware-lab.yar` |
| YARA Active Response | `/var/ossec/active-response/bin/yara-scan` |
| YARA result log | `/var/ossec/logs/yara-results.log` |
| Suricata EVE JSON | `/var/log/suricata/eve.json` |
| Wazuh FIM database | `/var/ossec/queue/fim/db/fim.db` |

---

# 24. Detection Layers

The lab intentionally uses multiple security layers.

```
                    SOC Visibility
                         │
        ┌────────────────┼────────────────┐
        │                │                │
   Endpoint           Network          File Hunting
        │                │                │
      Wazuh          Suricata /          YARA
      FIM               Zeek              │
        │                │                │
        └────────────────┼────────────────┘
                         │
                   Wazuh Manager
                         │
                 Rules / Correlation
                         │
                    Dashboard
```

### Why multiple tools?

They answer different questions.

**Suricata:**
> What suspicious network traffic/signature was observed?

**Zeek:**
> What was happening in the network at the protocol/connection level?

**Wazuh FIM:**
> What changed on the endpoint?

**YARA:**
> Does this file contain characteristics matching a detection rule?

**Wazuh Manager:**
> How do we centralize, correlate, alert, and investigate the evidence?

**Dashboard:**
> How does the analyst see and investigate the result?

---

# 25. What We Have Actually Built

The lab now demonstrates several complete detection chains.

### SSH

```
SSH attempt
 ↓
sshd
 ↓
journald
 ↓
Wazuh Agent
 ↓
Wazuh Manager
 ↓
SSH rules
 ↓
Dashboard
```

### Network reconnaissance

```
Nmap
 ↓
TCP SYN traffic
 ↓
Suricata
 ↓
eve.json
 ↓
Wazuh
 ↓
Alert
 ↓
Dashboard
```

### Zeek telemetry

```
Network activity
 ↓
Zeek
 ↓
Structured logs
 ↓
Ingestion pipeline
 ↓
Wazuh
 ↓
Detection
 ↓
Dashboard
```

### File integrity

```
File change
 ↓
Wazuh FIM
 ↓
554 / 550 / 553
 ↓
Dashboard
```

### Malware-test + YARA

```
File change
 ↓
Wazuh FIM
 ↓
Active Response
 ↓
YARA
 ↓
YARA_MATCH
 ↓
Wazuh log collection
 ↓
Rule 100500
 ↓
Level 12 alert
 ↓
Dashboard
```

---

# 26. Learning Order

The practical learning order used in this lab was:

1. Wazuh fundamentals
2. SSH authentication telemetry
3. SSH brute-force correlation
4. Suricata network detection
5. Nmap-generated reconnaissance
6. Zeek telemetry and Wazuh integration
7. Wazuh FIM
8. EICAR investigation
9. YARA fundamentals through practical experiments
10. YARA → Wazuh Active Response integration
11. **Next: formal YARA language from scratch**

---

# 27. Current Status

### Completed

- Wazuh core lab
- SSH monitoring
- SSH brute-force detection
- Suricata integration
- Nmap reconnaissance testing
- Zeek integration
- FIM
- EICAR investigation
- YARA practical toolbox
- YARA → Wazuh automation
- Dashboard verification

### Next

Formal YARA language learning:

```
Rule structure
     ↓
Identifiers
     ↓
Strings
     ↓
String modifiers
     ↓
Conditions
     ↓
Boolean logic
     ↓
Modules
     ↓
Helper rules
     ↓
Write rules from scratch
     ↓
Positive / negative testing
```

> This page is the lab's **Tools Corner**: use it whenever you need to understand what a tool does, why it exists, and how it connects to the rest of the SOC pipeline.
