# Tools Corner

This is the lab's reference page for understanding **what each tool does, where it runs, and how it connects to Wazuh**.

The goal is to understand the system, not memorize commands.

## 1. Core stack

| Tool | Role | Runs on | Wazuh connection |
|---|---|---|---|
| Wazuh Manager | Analysis, rules, alerts, correlation | Docker | Central |
| Wazuh Agent | Endpoint collection and response | Arch host | Direct |
| Wazuh Indexer | Event storage/search | Docker | Backend |
| Wazuh Dashboard | SOC investigation UI | Docker | Reads alerts |
| Docker | Wazuh deployment layer | Arch host | Hosts Manager/Indexer/Dashboard |
| journald | Host log source | Arch host | Agent collects it |
| sshd | Authentication source | Arch host | journald → Agent |
| FIM/syscheckd | File-change detection | Arch host | Agent → Manager |
| Suricata | Network IDS | Arch host | JSON → Wazuh |
| Zeek | Network telemetry | Arch host | Telemetry → Wazuh |
| YARA | File/content hunting | Arch host | Active Response |
| Nmap | Controlled network test | Test client | Traffic → Suricata |
| VirusTotal | IOC investigation | External | Analyst workflow |
| jq | JSON parsing | Arch host | YARA response script |
| SQLite | FIM DB inspection | Arch host | Investigation |
| Termux | Controlled test client | Test device | Generates lab traffic |

## 2. Wazuh

Think of Wazuh as the **central SOC platform**.

It does not replace every security tool.

It receives telemetry from other layers, applies rules/correlation, stores the resulting events, and presents them to the analyst.

### Important rules used

| Rule | Meaning in this lab |
|---:|---|
| 5710 | SSH invalid user |
| 5760 | SSH authentication failure |
| 5763 | Built-in SSH brute-force correlation |
| 100003 | Custom SSH brute-force correlation |
| 554 | FIM file added |
| 550 | FIM file modified |
| 553 | FIM file deleted |
| 86601 | Suricata alert ingestion |
| 100500 | YARA result converted into Wazuh alert |

## 3. Journald + sshd

```
sshd
 ↓
systemd journal
 ↓
Wazuh Agent
 ↓
Wazuh Manager
```

The important lesson is that the Manager does not directly depend on the host journal. The endpoint Agent collects the journal and forwards the relevant events.

## 4. FIM

FIM answers:

> Did a monitored file appear, change, or disappear?

The lab monitors:

```
/opt/wazuh-malware-lab
```

in realtime.

FIM is the **trigger** for the YARA workflow. It is not the malware classifier.

## 5. YARA

YARA answers:

> Does this file match a rule describing an indicator or pattern?

YARA is both a rule language and an engine that evaluates those rules.

Version:

```
4.5.6
```

The lab first tested YARA independently, then connected it to Wazuh.

## 6. Suricata

Suricata answers:

> Does network traffic match one of the IDS signatures?

Main output:

```
/var/log/suricata/eve.json
```

Interface used:

```
wlp8s0
```

Custom SIDs:

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
| 1000009 | DNS |
| 1000010 | FIN |
| 1000011 | NULL |
| 1000012 | Xmas |

## 7. Zeek

Zeek answers a different question:

> What structured network activity happened?

It produces telemetry around connections and protocols such as DNS, HTTP, and SSL/TLS.

The tested ingestion pipeline maps Zeek's native `id` value to:

```
data.zeek_id
```

to avoid the field-mapping conflict encountered during testing.

## 8. Nmap

Nmap is a **test generator**, not the detection system.

For example:

```
Nmap
 ↓
TCP SYN traffic
 ↓
Suricata
 ↓
Wazuh
```

The lab uses Nmap only against authorized lab targets.

## 9. VirusTotal

VirusTotal is used after an IOC such as a SHA-256 has been extracted.

Workflow:

```
Wazuh FIM
 ↓
SHA-256
 ↓
VirusTotal lookup
 ↓
Vendor labels / file identity
 ↓
Analyst conclusion
```

For the EICAR lab, the result was interpreted as a known antivirus test artifact, not as a real infection.

## 10. YARA → Wazuh connection

The important chain is:

```
FIM event
 ↓
Rule 554 / 550
 ↓
Active Response: yara-scan
 ↓
YARA rule evaluation
 ↓
YARA_MATCH
 ↓
/var/ossec/logs/yara-results.log
 ↓
Wazuh log collection
 ↓
Rule 100500
 ↓
Dashboard
```

The scanner is intentionally restricted to:

```
/opt/wazuh-malware-lab/*
```

This keeps the learning workflow controlled.

## 11. Important files

```
/opt/yara-rules/wazuh-malware-lab.yar
/var/ossec/active-response/bin/yara-scan
/var/ossec/logs/yara-results.log
/opt/wazuh-malware-lab/
/var/log/suricata/eve.json
/var/ossec/queue/fim/db/fim.db
```

## 12. Important ports

```
1514-1515/tcp  Wazuh agent communication
514/udp         Wazuh event input
55000/tcp       Wazuh API
9200            Wazuh Indexer
443 → 5601      Wazuh Dashboard
```

## 13. Detection layers

```
Source
 ↓
Telemetry
 ↓
Detection engine
 ↓
Wazuh collection
 ↓
Wazuh rule/correlation
 ↓
Storage
 ↓
Dashboard
 ↓
Analyst investigation
```

The same model applies repeatedly across the lab.

## 14. Learning order

The practical learning sequence is:

1. Wazuh basics
2. Host authentication
3. SSH detection and correlation
4. FIM
5. EICAR and IOC investigation
6. Suricata
7. Zeek
8. YARA language
9. YARA → Wazuh automation
10. Future Windows/Sysmon work

