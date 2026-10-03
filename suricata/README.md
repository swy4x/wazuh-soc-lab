# Suricata Detection Layer

Suricata is the network IDS layer of this lab.

## Role

Suricata inspects network traffic and produces structured detection events.

It is not the SIEM. Wazuh receives and processes the resulting telemetry.

## Environment

Interface:

```
wlp8s0
```

Main log:

```
/var/log/suricata/eve.json
```

## Detection flow

```
Network traffic
 ↓
Suricata
 ↓
Signature
 ↓
eve.json
 ↓
Wazuh
 ↓
Wazuh rule / correlation
 ↓
Dashboard
```

## Tested behaviors

- TCP SYN scan / port sweep
- UDP scan / burst
- DNS reconnaissance
- ICMP reconnaissance
- TCP FIN scan
- TCP NULL scan
- TCP Xmas scan

Keep Suricata detection IDs and Wazuh rule IDs conceptually separate.
