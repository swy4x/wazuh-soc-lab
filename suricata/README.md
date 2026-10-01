# Suricata Detection Layer

Suricata is the network IDS layer in this lab.

## Log source

`/var/log/suricata/eve.json`

## Tested detections

- TCP SYN scan / port sweep
- UDP scan / burst
- DNS reconnaissance
- TCP FIN scan
- TCP NULL scan
- TCP Xmas scan
- ICMP reconnaissance

## Detection model

Suricata detects the network behavior and records structured JSON. Wazuh then ingests those events and applies its own rules.

Keep the two layers conceptually separate:

```
Suricata signature → network detection
Wazuh rule         → SIEM ingestion/correlation/alerting
```

Custom rule files should be added here only after they are exported from the working lab and verified.
