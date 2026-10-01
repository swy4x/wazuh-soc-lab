# Nmap TCP SYN Scan Detection

## Objective

Generate a controlled TCP SYN scan against the lab host and verify that Suricata and Wazuh detect it.

## Test

The scanner used Nmap with a TCP SYN scan against the lab host:

```bash
nmap -sS -p 1-1000 <LAB_HOST_IP>
```

Only authorized lab systems should be scanned.

## Observed Wazuh event

Example observed fields:

| Field | Value |
|---|---|
| Suricata signature | `LAB: TCP SYN Scan / Port Sweep` |
| Suricata signature ID | `1000001` |
| Protocol | TCP |
| Source IP | `10.170.114.183` |
| Destination IP | `10.170.114.88` |
| Destination port | `334` |
| Interface | `wlp8s0` |
| Event type | `alert` |
| Wazuh rule | `86601` |
| Wazuh level | `3` |

## Investigation

The source and destination fields allow the analyst to establish:

- `10.170.114.183` = scanner/source
- `10.170.114.88` = lab host/target
- TCP was used
- Suricata classified the activity as a TCP SYN scan / port sweep

## Important note

Wazuh rule `86601` is the generic Suricata alert ingestion rule in this lab. The more specific detection identity comes from the Suricata signature and signature ID.

## Next improvement

Create higher-level Wazuh correlation rules for reconnaissance and investigate whether network characteristics can distinguish likely scanning tools such as Nmap or Masscan. Tool identification should be treated as a fingerprint/likelihood, not guaranteed attribution from packets alone.
