# Suricata Local Rules

These are custom detection rules used in the Wazuh SOC lab.

> Test only against systems and networks you own or are explicitly authorized to assess.

## Rules

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

## Tested rules

### SID 1000001 — TCP SYN Scan / Port Sweep

Triggered during a controlled Nmap TCP SYN scan. The resulting Suricata alert was ingested by Wazuh under rule 86601.

### SID 1000008 — UDP Scan / Burst

Observed in the Wazuh Dashboard with the signature:

`LAB: UDP Scan / Burst`

The event showed UDP traffic associated with DNS.

## Notes

The working lab rules are intentionally kept simple for learning. Thresholds and packet characteristics should be tuned to reduce false positives before production use.
