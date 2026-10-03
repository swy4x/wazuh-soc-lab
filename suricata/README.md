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


## Custom Signature Catalogue

The lab used the following custom Suricata SIDs:

\`\`\`
1000001  TCP SYN Scan / Port Sweep
1000002  ICMP Recon / Ping Sweep
1000003  SSH Connection Burst
1000004  SMB Scan
1000005  RDP Scan
1000006  HTTP Port Scan
1000007  HTTPS Port Scan
1000008  UDP Scan / Burst
1000009  DNS Query Burst
1000010  TCP FIN Scan
1000011  TCP NULL Scan
1000012  TCP Xmas Scan
\`\`\`

The Nmap validation proved the complete path for SID 1000001:

\`\`\`
Nmap
 ↓
TCP SYN traffic
 ↓
Suricata SID 1000001
 ↓
/var/log/suricata/eve.json
 ↓
Wazuh Rule 86601
 ↓
Dashboard
\`\`\`

The repository records the verified SID catalogue and observed behavior. It does not currently contain the original Suricata signature source, so exact rule bodies are not reconstructed.
