# Nmap TCP SYN Scan Detection

## Objective

Generate a controlled TCP SYN scan and verify that Suricata detects it and Wazuh receives the resulting alert.

## Test

```bash
nmap -sS -p 1-1000 <LAB_HOST_IP>
```

Only authorized lab systems should be scanned.

## Observed event

| Field | Value |
|---|---|
| Suricata signature | LAB: TCP SYN Scan / Port Sweep |
| Suricata SID | 1000001 |
| Protocol | TCP |
| Source IP | 10.170.114.183 |
| Destination IP | 10.170.114.88 |
| Destination port | 334 |
| Interface | wlp8s0 |
| Event type | alert |
| Wazuh rule | 86601 |
| Wazuh level | 3 |

## Detection flow

```
Nmap
 ↓
TCP SYN traffic
 ↓
Suricata
 ↓
SID 1000001
 ↓
eve.json
 ↓
Wazuh
 ↓
Rule 86601
 ↓
Dashboard
```

## Important distinction

Suricata's SID and Wazuh's rule ID represent different layers.

- **1000001** = the Suricata network detection.
- **86601** = Wazuh's observed Suricata alert-ingestion rule.

The Wazuh rule does not replace the Suricata signature.


## Custom Suricata Rule

The controlled scan was detected by the lab's custom Suricata SID 1000001:

```
LAB: TCP SYN Scan / Port Sweep
SID: 1000001
```

The event was then ingested by Wazuh under Rule 86601.

The current repository stores the verified SID and observed event but not the original Suricata signature body. The documentation therefore does not reconstruct the missing source syntax.

```
Nmap
  ↓
TCP SYN traffic
  ↓
Suricata SID 1000001
  ↓
eve.json
  ↓
Wazuh Rule 86601
  ↓
Dashboard
```
