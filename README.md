# Wazuh SOC Lab

A hands-on SOC lab built around Wazuh, endpoint telemetry, network monitoring, file integrity monitoring, and controlled detection experiments.

The lab is built on an Arch Linux endpoint with the Wazuh stack running in Docker. The goal is to understand **how security telemetry moves from a source to a SOC alert**, not just to collect tools.

## Lab stack

- Wazuh Manager 4.14.1
- Wazuh Agent 4.14.5
- Wazuh Indexer
- Wazuh Dashboard
- Docker
- systemd journald
- sshd
- Suricata 8.x
- Zeek
- YARA 4.5.6
- Nmap for controlled reconnaissance testing
- VirusTotal for IOC investigation

## Architecture

### Host authentication

```
sshd
  ↓
systemd journald
  ↓
Wazuh Agent
  ↓
Wazuh Manager
  ↓
Wazuh rules / correlation
  ↓
Wazuh Indexer
  ↓
Wazuh Dashboard
```

### Network monitoring

```
Network traffic
  ├──→ Suricata → eve.json → Wazuh
  └──→ Zeek → structured telemetry → Wazuh
```

Suricata and Zeek are deliberately treated as different layers:

- **Suricata** — signature-based network IDS.
- **Zeek** — structured network-security telemetry.
- **Wazuh** — centralized collection, rules, correlation, storage, and investigation.

### FIM → YARA → Wazuh

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
Wazuh-collected result log
  ↓
Rule 100500
  ↓
Dashboard alert
```

## Completed labs

### Endpoint

- Journald collection
- SSH invalid-user detection
- SSH authentication-failure correlation
- Custom SSH brute-force rule
- Realtime FIM
- File creation, modification, and deletion detection
- EICAR test-artifact detection
- SHA-256 IOC investigation

### Network

- Suricata → Wazuh
- Controlled TCP SYN scan detection
- UDP, DNS, ICMP, FIN, NULL, and Xmas test rules
- Zeek → Wazuh
- Zeek SSL telemetry
- Zeek DNS reconnaissance correlation
- Zeek `id` field mapping fix using `data.zeek_id`

### YARA

- String matching
- `any of them`
- `all of them`
- Boolean AND / OR / NOT
- File-size conditions
- Hash matching
- Hash vs content comparison
- Regex
- Metadata
- Private helper rules
- Tags
- ELF module
- PE module
- PE architecture
- PE + indicator conditions
- Recursive hunting
- YARA → Wazuh Active Response integration

## Important lab rules

- Testing is performed only against systems and networks controlled for the lab.
- EICAR is used as a harmless antivirus test artifact and is never executed.
- The YARA Active Response scanner is restricted to the lab directory.
- Screenshots are added manually during the learning/documentation phase.

## Repository structure

```
README.md
docs/
  architecture.md
  tools-corner.md
  yara-practical.md
  yara-wazuh-integration.md
attacks/
  ssh-invalid-user.md
  ssh-brute-force.md
  nmap-tcp-syn-scan.md
  eicar-fim-detection.md
investigations/
  eicar-hash-investigation.md
network/
  zeek-wazuh.md
suricata/
  README.md
  local-rules.md
```

## Start here

1. Read [Tools Corner](docs/tools-corner.md) for the complete tool map.
2. Read [Architecture](docs/architecture.md) for the system design.
3. Use the attack and investigation notes as evidence of completed labs.
4. Use [YARA Practical](docs/yara-practical.md) for the YARA learning sequence.
5. Use [YARA → Wazuh Integration](docs/yara-wazuh-integration.md) for the final automated workflow.

> This repository documents the lab as actually built and tested. It does not claim that a lab rule is production-ready merely because it worked in testing.
