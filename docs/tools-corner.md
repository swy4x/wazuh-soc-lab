# Tools Corner — A–Z Lab Reference

> **The one page to understand what each tool does, where it runs, why the lab uses it, and how it connects to Wazuh.**

The lab is intentionally built from specialized layers.

Wazuh is the central SOC platform, but it does **not** replace the endpoint, network, file-analysis, or investigation tools around it.

---

# 1.  The Complete Stack

| Component | Job | Runs on | Output / Connection |
|---|---|---|---|
| **Wazuh Manager** | Decode, analyze, correlate, alert, coordinate response | Docker | Central analysis |
| **Wazuh Agent** | Collect endpoint telemetry + execute selected response | Arch host | → Manager |
| **Wazuh Indexer** | Store/search Wazuh data | Docker | ← Manager |
| **Wazuh Dashboard** | Analyst investigation interface | Docker | ← Indexer |
| **Docker** | Runs Wazuh stack | Arch host | Hosts Wazuh |
| **systemd journald** | Host log storage | Arch host | → Agent |
| **sshd** | SSH authentication source | Arch host | → journald |
| **FIM / syscheckd** | File change detection | Arch host | → Agent |
| **Suricata** | Network IDS/signatures | Arch host | → Wazuh |
| **Zeek** | Structured network telemetry | Arch host | → Wazuh |
| **YARA** | File/content detection | Arch host | Active Response |
| **Nmap** | Controlled traffic generator | Test client | → Suricata |
| **VirusTotal** | IOC investigation | External | Analyst |
| **jq** | JSON parsing | Arch host | YARA response script |
| **SQLite** | FIM database inspection | Arch host | Analyst |
| **Termux** | Controlled test client | Test device | Generates traffic |

---

# 2.  Wazuh — The Central SOC Layer

Think of Wazuh as the **central nervous system** of the lab.

It receives telemetry from different sources and turns that telemetry into alerts and searchable investigation data.

### Wazuh handles

- collection/ingestion;
- decoding;
- rules;
- correlation;
- alert generation;
- Active Response orchestration;
- storage through the Indexer;
- analyst visibility through Dashboard.

### Wazuh does NOT become

- Suricata;
- Zeek;
- YARA;
- sshd;
- Nmap.

Each has a separate role.

### Important lab rules

| Rule | Meaning |
|---:|---|
| 5710 | SSH invalid user |
| 5760 | SSH authentication failure |
| 5763 | Built-in SSH brute-force correlation |
| 100003 | Custom SSH brute-force correlation |
| 554 | FIM file added |
| 550 | FIM file modified |
| 553 | FIM file deleted |
| 86601 | Suricata alert ingestion |
| 100500 | YARA result converted to Wazuh alert |

---

# 3.  Wazuh Agent

The Agent runs on the Arch endpoint.

In this lab it has two major jobs.

### Collection

```
journald
FIM
YARA result log
```

### Response

```
Active Response → yara-scan
```

Agent:

```
ID:       001
Name:     archlinux
Manager:  127.0.0.1
Version:  4.14.5
```

---

# 4.  Wazuh Manager

Container:

```
single-node-wazuh.manager-1
```

Version:

```
4.14.1
```

Its job is to turn raw telemetry into security meaning.

Conceptually:

```
raw event
   ↓
decoder
   ↓
rule
   ↓
correlation
   ↓
alert
```

---

# 5.  Wazuh Indexer

The Indexer is the searchable storage layer.

Lab port:

```
9200
```

It stores the alert/event data that the Dashboard presents to the analyst.

---

# 6.  Wazuh Dashboard

The Dashboard is where we investigate the results.

Lab exposure:

```
443 → 5601
```

It was used to verify:

- SSH alerts;
- brute-force alerts;
- Suricata alerts;
- FIM events;
- YARA Rule 100500.

---

# 7.  Docker

Docker is the **deployment layer**, not a detection tool.

The Wazuh stack lives at:

```
~/wazuh-docker/single-node
```

Containers:

```
single-node-wazuh.manager-1
single-node-wazuh.dashboard-1
single-node-wazuh.indexer-1
```

---

# 8.  systemd journald

journald is the host's log source.

The flow is:

```
sshd
 ↓
journald
 ↓
Wazuh Agent
 ↓
Wazuh Manager
```

Agent configuration:

```xml
<localfile>
  <log_format>journald</log_format>
  <location>journald</location>
</localfile>
```

### Important concept

The Manager is not directly reading the Arch host's journal.

The **Agent** reads it and forwards the relevant telemetry.

---

# 9.  sshd

sshd produces authentication events.

the lab tested:

- invalid usernames;
- wrong passwords;
- repeated authentication failures.

Important rules:

```
5710 → invalid user
5760 → authentication failure
5763 → built-in brute-force correlation
100003 → custom lab correlation
```

---

# 10.  SSH Correlation — Rule 100003

The custom rule:

```xml
<rule id="100003" level="10" frequency="3" timeframe="60" ignore="60">
  <if_matched_sid>5760</if_matched_sid>
  <same_source_ip/>
  <description>LAB: SSH Brute Force - 3 authentication failures from the same source IP within 60 seconds</description>
  <mitre>
    <id>T1110</id>
  </mitre>
  <group>authentication_failed,brute_force,ssh,lab,mitre_t1110,</group>
</rule>
```

Meaning:

- `if_matched_sid 5760` → authentication failure is the input;
- `frequency 3` → three failures;
- `timeframe 60` → within 60 seconds;
- `same_source_ip` → same attacker/source;
- `ignore 60` → suppress repeated firing for 60 seconds;
- `T1110` → MITRE ATT&CK Brute Force.

This is a real behavioral correlation example.

---

# 11.  FIM / syscheckd

FIM answers:

> **Did a monitored file appear, change, or disappear?**

Realtime directory:

```
/opt/wazuh-malware-lab
```

Pipeline:

```
file event
 ↓
inotify
 ↓
wazuh-syscheckd
 ↓
Wazuh Agent
 ↓
Wazuh Manager
 ↓
FIM rule
 ↓
alert
```

Rules:

| Rule | Meaning |
|---:|---|
| 554 | File added |
| 550 | File modified |
| 553 | File deleted |

Database:

```
/var/ossec/queue/fim/db/fim.db
```

The lab also verified:

- inotify watch limits;
- inotify instance limits;
- queued events;
- `wazuh-syscheckd` using `anon_inode:inotify`.

---

# 12.  EICAR

EICAR is a standardized antivirus test artifact.

Path:

```
/opt/wazuh-malware-lab/eicar.com
```

It was never executed.

It gives us a deterministic and harmless object for:

```
FIM
 ↓
hash investigation
 ↓
YARA
 ↓
Wazuh automation
```

---

# 13.  Suricata

Suricata is the **network IDS/signature engine**.

Interface:

```
wlp8s0
```

Output:

```
/var/log/suricata/eve.json
```

Suricata answers:

> **Does this traffic match one of my detection signatures?**

It is not the SIEM.

---

# 14.  Suricata Local SIDs

| SID | Detection |
|---:|---|
| 1000001 | TCP SYN scan |
| 1000002 | ICMP |
| 1000003 | SSH burst |
| 1000004 | SMB |
| 1000005 | RDP |
| 1000006 | HTTP |
| 1000007 | HTTPS |
| 1000008 | UDP |
| 1000009 | DNS burst |
| 1000010 | TCP FIN |
| 1000011 | TCP NULL |
| 1000012 | TCP Xmas |

Validated chain:

```
Nmap
 ↓
TCP SYN traffic
 ↓
Suricata SID 1000001
 ↓
eve.json
 ↓
Wazuh
 ↓
Rule 86601
 ↓
Dashboard
```

---

# 15.  Nmap

Nmap is a **controlled test generator**.

It does not detect the scan.

It creates the traffic needed to prove that Suricata can detect it.

```
Nmap
 ↓
network traffic
 ↓
Suricata
 ↓
Wazuh
```

Only authorized targets are used.

---

# 16.  Zeek

Zeek provides structured network-security telemetry.

A useful separation:

| Tool | Question |
|---|---|
| **Suricata** | Does traffic match a detection signature? |
| **Zeek** | What structured network activity happened? |
| **Wazuh** | What should the SOC collect/correlate/alert on? |

Tested rules:

| Rule | Level | Purpose |
|---:|---:|---|
| 100102 | 3 | Synthetic SSL |
| 100304 | 7 | DNS reconnaissance |
| 100305 | 10 | DNS reconnaissance correlation |

---

# 17.  Zeek Mapping Bug

Zeek has a native:

```
id
```

field.

The ingestion pipeline encountered a field-mapping conflict.

The working mapping uses:

```
data.zeek_id
```

This is an important real-world lesson:

> Integrations can fail because two systems assign different meanings or structures to the same field.

---

# 18.  YARA

YARA has two closely related meanings.

### YARA language

The syntax used to write detection rules.

### YARA engine

The `yara` executable that evaluates those rules.

So:

```
.yar file
   ↓
YARA language
   ↓
yara engine
   ↓
match / no match
```

Version:

```
4.5.6
```

The practical phase tested strings, Boolean logic, hashes, regex, metadata, helper rules, tags, ELF, PE, and recursive hunting.

---

# 19.  YARA Practical Toolbox

Tested capabilities:

1. literal string matching;
2. `any of them`;
3. `all of them`;
4. AND;
5. OR;
6. NOT;
7. file-size conditions;
8. SHA-256 matching;
9. hash vs content behavior;
10. regular expressions;
11. metadata;
12. private helper rules;
13. tags;
14. ELF module;
15. PE module;
16. PE architecture;
17. PE + indicator logic;
18. recursive hunting.

Detailed record:

 `docs/yara-practical.md`

---

# 20.  YARA → Wazuh

The completed integration:

```
FIM
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
yara-results.log
 ↓
Wazuh Agent
 ↓
Rule 100500
 ↓
Dashboard
```

Important files:

```
/opt/yara-rules/wazuh-malware-lab.yar
/var/ossec/active-response/bin/yara-scan
/var/ossec/logs/yara-results.log
```

---

# 21.  YARA Active Response — Important Debugging

The first scanner used:

```bash
INPUT="$(cat)"
```

It hung because `cat` waited for EOF while Wazuh kept the Active Response input pipe open.

The process was observed waiting on a pipe.

The working approach:

```bash
IFS= read -r INPUT
```

This reads one event and allows the process to exit normally.

This mistake is intentionally documented because understanding **why** it failed is more valuable than pretending the first implementation worked.

---

# 22.  jq

The Active Response event is JSON.

`jq` extracts:

- the command;
- the FIM path.

Conceptually:

```
Wazuh JSON
 ↓
jq
 ↓
command + file path
 ↓
YARA
```

---

# 23.  SQLite

The FIM database was inspected with SQLite:

```
/var/ossec/queue/fim/db/fim.db
```

This connected the conceptual idea of:

```
baseline
+
hashes
+
metadata
```

with the actual endpoint database.

---

# 24.  VirusTotal

VirusTotal is used for **IOC investigation**, not as part of Wazuh's detection engine.

Workflow:

```
Wazuh FIM
 ↓
SHA-256
 ↓
VirusTotal
 ↓
reputation / labels
 ↓
analyst interpretation
```

For EICAR, the external result was interpreted correctly as a known antivirus test artifact.

---

# 25.  Termux

Termux was used as a convenient controlled test client for network experiments.

It can generate traffic toward the lab so that Suricata/Zeek telemetry can be observed.

---

# 26.  Important Ports

```
1514-1515/tcp  → Wazuh Agent communication
514/udp         → Wazuh event input
55000/tcp       → Wazuh API
9200            → Wazuh Indexer
443 → 5601      → Wazuh Dashboard
```

---

# 27.  Important Paths

```
~/wazuh-docker/single-node

/opt/wazuh-malware-lab
/opt/wazuh-malware-lab/eicar.com

/opt/yara-rules/wazuh-malware-lab.yar

/var/ossec/active-response/bin/yara-scan
/var/ossec/logs/yara-results.log

/var/ossec/queue/fim/db/fim.db

/var/log/suricata/eve.json
```

---

# 28.  Detection-Layer Mental Model

```
                  SECURITY ACTIVITY
                         ↓
              ┌──────────┴──────────┐
              │                     │
           Endpoint               Network
              │                     │
        ┌─────┴─────┐          ┌────┴────┐
        │           │          │         │
     journald      FIM      Suricata    Zeek
        │           │          │         │
       sshd       files        │         │
                    │          │         │
                    └────┬─────┴─────────┘
                         │
                       Wazuh
                         │
                    correlation
                         │
                       alert
                         │
                     Dashboard
                         │
                    investigation
```

YARA is an additional **file-analysis layer** invoked from the FIM path.

---

# 29.  Learning Order

The lab's learning progression:

```
1. Wazuh foundation
       ↓
2. Agent + journald
       ↓
3. SSH detection
       ↓
4. SSH correlation
       ↓
5. FIM
       ↓
6. EICAR
       ↓
7. IOC investigation
       ↓
8. Suricata
       ↓
9. Nmap testing
       ↓
10. Zeek
       ↓
11. YARA practical
       ↓
12. YARA → Wazuh
       ↓
13. Formal YARA language
```

The objective is to understand the role of each layer rather than memorize isolated commands.

---

# 30.  Current Status

### Completed

 Wazuh endpoint foundation  
 journald collection  
 SSH invalid-user detection  
 SSH brute-force correlation  
 realtime FIM  
 EICAR investigation  
 hash/IOC investigation  
 Suricata  
 Nmap → Suricata validation  
 Zeek → Wazuh  
 practical YARA  
 YARA → Wazuh automation  

### Current learning phase

**Formal YARA language from zero.**

---

> **Tools Corner should grow whenever a new tool is added. The page exists so the lab never becomes a pile of disconnected commands.**


# 31. Custom Detection Rules

## Wazuh Rule 100003

```xml
<rule id="100003" level="10" frequency="3" timeframe="60" ignore="60">
  <if_matched_sid>5760</if_matched_sid>
  <same_source_ip/>
  <description>LAB: SSH Brute Force - 3 authentication failures from the same source IP within 60 seconds</description>
  <mitre><id>T1110</id></mitre>
  <group>authentication_failed,brute_force,ssh,lab,mitre_t1110,</group>
</rule>
```

## Wazuh Rule 100500

```xml
<rule id="100500" level="12">
  <match>YARA_MATCH</match>
  <description>YARA detected a malware-test indicator in a FIM-monitored file</description>
  <group>yara,malware_detection,file_integrity,lab,</group>
</rule>
```

## YARA Rule Wazuh_EICAR_Test

```yara
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
```

## Network rule catalogue

Suricata SIDs used:

| SID | Detection |
|---:|---|
| 1000001 | TCP SYN Scan / Port Sweep |
| 1000002 | ICMP Recon / Ping Sweep |
| 1000003 | SSH Connection Burst |
| 1000004 | SMB Scan |
| 1000005 | RDP Scan |
| 1000006 | HTTP Port Scan |
| 1000007 | HTTPS Port Scan |
| 1000008 | UDP Scan / Burst |
| 1000009 | DNS Query Burst |
| 1000010 | TCP FIN Scan |
| 1000011 | TCP NULL Scan |
| 1000012 | TCP Xmas Scan |

Zeek/Wazuh rules:

| Rule | Level | Purpose |
|---:|---:|---|
| 100102 | 3 | Synthetic Zeek SSL |
| 100304 | 7 | DNS reconnaissance |
| 100305 | 10 | DNS reconnaissance correlation |

The current repository stores the verified Suricata and Zeek identifiers and observed behavior, but not the original source bodies. The documentation intentionally does not invent missing rule syntax.