# Wazuh SOC Lab

 > **A documented security monitoring and detection lab implemented on Arch Linux.**
>
> **Goal:** understand the complete path from **activity → telemetry → detection → correlation → alert → investigation**.

---

## What this repository is

This repository documents a deliberately layered security monitoring environment.

Each major component was installed, configured, validated, and integrated with the surrounding detection pipeline. The documentation records the implemented configuration, observed results, validation steps, and relevant engineering decisions.

The lab currently covers:

-  Wazuh SIEM/XDR fundamentals
-  SSH authentication detection and brute-force correlation
-  File Integrity Monitoring
-  EICAR-based malware-test investigation
-  Suricata network IDS
-  Zeek network telemetry
-  YARA file/content detection
-  YARA → Wazuh Active Response automation
-  IOC/hash investigation

---

# Lab Environment

### Endpoint

| Component | Value |
|---|---|
| OS | Arch Linux |
| Shell | Fish |
| Network interface | `wlp8s0` |
| Wazuh Agent | 4.14.5-1 |
| Wazuh Manager | 4.14.1 |
| Suricata | 8.0.4-2 |
| YARA | 4.5.6 |

### Wazuh deployment

The Wazuh single-node stack runs in Docker:

```
~/wazuh-docker/single-node
```

Containers:

```
single-node-wazuh.manager-1
single-node-wazuh.dashboard-1
single-node-wazuh.indexer-1
```

### Ports

| Port | Purpose |
|---|---|
| 1514-1515/tcp | Wazuh Agent communication |
| 514/udp | Event input |
| 55000/tcp | Wazuh API |
| 9200 | Wazuh Indexer |
| 443 → 5601 | Wazuh Dashboard |

---

# The Big Picture

The lab is built as several detection layers around Wazuh.

```
                         ┌────────────────────┐
                         │  Wazuh Dashboard   │
                         └─────────▲──────────┘
                                   │
                         ┌─────────┴──────────┐
                         │   Wazuh Indexer    │
                         └─────────▲──────────┘
                                   │
                         ┌─────────┴──────────┐
                         │   Wazuh Manager    │
                         │ Rules • Correlation│
                         │ Alerts • Response  │
                         └───────┬─┬─┬────────┘
                                 │ │ │
                    ┌────────────┘ │ └────────────┐
                    │              │              │
             Wazuh Agent         Zeek         Suricata
                    │              │              │
             ┌──────┼──────┐       │              │
             │      │      │       │              │
          journald  FIM   YARA   network        network
             │      │
            sshd    files
```

The important separation is:

| Layer | Main question |
|---|---|
| **sshd/journald** | What happened on the host? |
| **FIM** | What changed on the filesystem? |
| **Suricata** | Did traffic match an IDS signature? |
| **Zeek** | What structured network activity happened? |
| **YARA** | Does this file match my detection logic? |
| **Wazuh** | How do we collect, correlate, alert, store, and investigate it? |

---

# Host Authentication

## Detection chain

```
SSH attempt
   ↓
sshd
   ↓
systemd journald
   ↓
Wazuh Agent
   ↓
Wazuh Manager
   ↓
Decoder + Rules
   ↓
Indexer
   ↓
Dashboard
```

The Agent collects the host journal using:

```xml
<localfile>
  <log_format>journald</log_format>
  <location>journald</location>
</localfile>
```

### Important rules observed

| Rule | Meaning |
|---:|---|
| 5710 | SSH invalid/non-existent user |
| 5760 | SSH authentication failure |
| 5763 | Built-in SSH brute-force correlation |
| 100003 | Custom SSH brute-force correlation |
| 2502 | Repeated password failures |
| 5503 | PAM login failure |
| 5557 | Password check failure |

### Custom correlation

Rule **100003** requires:

- 3 matching SSH authentication failures
- from the same source IP
- within 60 seconds
- then suppresses repeated alerts for 60 seconds

This demonstrates the difference between a **single event** and a **behavioral detection**.

 Detailed lab: [SSH brute-force detection](attacks/ssh-brute-force.md)

---

# File Integrity Monitoring

FIM answers:

> **What changed on the monitored endpoint?**

Realtime monitoring was enabled for:

```
/opt/wazuh-malware-lab
```

### Detection chain

```
File create / modify / delete
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
          Alert
```

### Rules

| Rule | Meaning |
|---:|---|
| 554 | File added |
| 550 | File modified / checksum changed |
| 553 | File deleted |

The FIM database was also inspected:

```
/var/ossec/queue/fim/db/fim.db
```

The lab verified Linux inotify resources and confirmed that `wazuh-syscheckd` was using inotify.

---

# EICAR Investigation

The lab uses the standardized **EICAR antivirus test artifact**.

It is harmless and was **never executed**.

Path:

```
/opt/wazuh-malware-lab/eicar.com
```

Wazuh reported:

| Property | Value |
|---|---|
| Size | 68 bytes |
| MD5 | `44d88612fea8a8f36de82e1278abb02f` |
| SHA-1 | `3395856ce81f2b7382dee72602f798b642f14140` |
| SHA-256 | `275a021bbfb6489e54d471899f7db9d1663fc695ec2fe2a2c4538aabf651fd0f` |

The local SHA-256 calculation matched the Wazuh value.

The exact SHA-256 was then investigated in VirusTotal. The important SOC lesson was:

> **A high detection count does not automatically mean a real infection. File identity and context matter.**

 [EICAR FIM detection](attacks/eicar-fim-detection.md)  
 [EICAR hash investigation](investigations/eicar-hash-investigation.md)

---

# Suricata

Suricata is the lab's **network IDS/signature layer**.

Interface:

```
wlp8s0
```

Main event log:

```
/var/log/suricata/eve.json
```

### Custom signatures

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
| 1000010 | TCP FIN |
| 1000011 | TCP NULL |
| 1000012 | TCP Xmas |

A controlled Nmap SYN scan generated Suricata SID **1000001** and was ingested by Wazuh under rule **86601**.

> **Important:** Suricata SID and Wazuh rule ID are different layers.

 [Suricata documentation](suricata/README.md)  
 [Local Suricata rules](suricata/local-rules.md)  
 [Nmap SYN scan lab](attacks/nmap-tcp-syn-scan.md)

---

# Zeek

Zeek is the lab's **structured network-telemetry layer**.

The conceptual separation is:

```
Suricata → "Does traffic match my signature?"
Zeek     → "What structured network activity happened?"
Wazuh    → "What should the SOC collect, correlate and investigate?"
```

### Tested rules

| Rule | Level | Purpose |
|---:|---:|---|
| 100102 | 3 | Synthetic Zeek SSL event |
| 100304 | 7 | DNS reconnaissance |
| 100305 | 10 | DNS reconnaissance correlation |

A real ingestion problem was encountered because Zeek uses an `id` field. The working mapping changed this to:

```
data.zeek_id
```

 [Zeek → Wazuh lab](network/zeek-wazuh.md)

---

# YARA

YARA was learned in two separate phases.

### Phase 1 — Practical toolbox

the lab tested:

- literal strings
- `any of them`
- `all of them`
- AND / OR / NOT
- file size
- SHA-256
- hash vs content behavior
- regex
- metadata
- private helper rules
- tags
- ELF module
- PE module
- PE architecture
- PE + indicator logic
- recursive hunting

### Phase 2 — Integration

YARA was then connected to Wazuh through Active Response.

YARA is both:

1. a **rule language**;
2. an **engine/program** that evaluates those rules.

 [YARA practical experiments](docs/yara-practical.md)

---

# YARA → Wazuh Automation

The completed automation is:

```
File created / modified
        ↓
Wazuh FIM
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
Indexer
        ↓
Dashboard
```

### Important files

```
/opt/yara-rules/wazuh-malware-lab.yar
/var/ossec/active-response/bin/yara-scan
/var/ossec/logs/yara-results.log
```

The final validated output was:

```
YARA_MATCH rule=Wazuh_EICAR_Test path=/opt/wazuh-malware-lab/eicar.com
```

Wazuh generated Rule **100500**, level **12**.

A real implementation bug was also fixed: the first scanner used `cat` on stdin and waited for EOF. It was changed to read exactly one Active Response event with:

```
IFS= read -r INPUT
```

 [Complete YARA → Wazuh integration](docs/yara-wazuh-integration.md)

---

# Tools Corner

For an overview of the complete stack, see:

 **[Open Tools Corner](docs/tools-corner.md)**

It explains:

- what every tool is;
- why it exists;
- where it runs;
- what it produces;
- how it connects to Wazuh;
- important ports and paths;
- complete detection chains.

---

# Repository Map

```
README.md
│
├── docs/
│   ├── architecture.md
│   ├── tools-corner.md
│   ├── yara-practical.md
│   └── yara-wazuh-integration.md
│
├── attacks/
│   ├── ssh-invalid-user.md
│   ├── ssh-brute-force.md
│   ├── nmap-tcp-syn-scan.md
│   └── eicar-fim-detection.md
│
├── investigations/
│   └── eicar-hash-investigation.md
│
├── network/
│   └── zeek-wazuh.md
│
└── suricata/
    ├── README.md
    └── local-rules.md
```

---

# Recommended Learning Order

The lab was built in this order:

```
Wazuh basics
     ↓
Agent + journald
     ↓
SSH detection
     ↓
SSH correlation
     ↓
FIM
     ↓
EICAR
     ↓
IOC investigation
     ↓
Suricata
     ↓
Nmap testing
     ↓
Zeek
     ↓
YARA practical
     ↓
YARA → Wazuh
     ↓
Formal YARA language
```

The next learning stage is **formal YARA language from zero**.

The practical integration is already complete; now the goal is to understand exactly how to write YARA rules instead of only using them.

---

# Lab Safety

- Only controlled/authorized systems are tested.
- EICAR is used as a harmless test artifact and is never executed.
- The YARA Active Response scanner is restricted to the lab directory.
- The custom Suricata rules are learning signatures, not claims of production-ready detection.
- Screenshots are collected during the documentation/learning phase rather than cluttering setup instructions.

---

> **This repository records the lab as actually built, tested, debugged, and understood. Successful experiments, failed experiments, configuration mistakes, and their fixes are kept separate so the documentation remains honest.**



## Custom Detection Rules

The lab's custom detection logic is documented separately in **[Custom Detection Rules Reference](docs/custom-rules.md)**.

### Wazuh Rule 100003 — SSH brute-force correlation

```xml
<rule id="100003" level="10" frequency="3" timeframe="60" ignore="60">
  <if_matched_sid>5760</if_matched_sid>
  <same_source_ip/>
  <description>LAB: SSH Brute Force - 3 authentication failures from the same source IP within 60 seconds</description>
  <mitre><id>T1110</id></mitre>
  <group>authentication_failed,brute_force,ssh,lab,mitre_t1110,</group>
</rule>
```

### Wazuh Rule 100500 — YARA result alert

```xml
<rule id="100500" level="12">
  <match>YARA_MATCH</match>
  <description>YARA detected a malware-test indicator in a FIM-monitored file</description>
  <group>yara,malware_detection,file_integrity,lab,</group>
</rule>
```

### YARA rule — Wazuh_EICAR_Test

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

The verified custom Suricata SID catalogue and Zeek/Wazuh rule catalogue are documented in the dedicated reference.

### Custom rule documentation

The repository now includes:

```
docs/custom-rules.md
```

This file is the reference point for the custom Wazuh rules, YARA rules, Suricata SID catalogue, and Zeek/Wazuh rule catalogue used throughout the lab.