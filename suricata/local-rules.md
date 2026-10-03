# Suricata Local Rules

Custom Suricata signatures used for controlled learning tests.

Only use these rules against systems and networks owned or explicitly authorized for testing.

## Rule index

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

## Validated examples

### SID 1000001

Triggered during the controlled Nmap TCP SYN scan.

The event was written to Suricata's `eve.json` and later appeared in Wazuh under the Suricata ingestion rule `86601`.

### SID 1000008

Observed as a UDP burst associated with DNS traffic.

## Design note

These rules are learning rules, not production signatures. Thresholds and traffic characteristics should be tuned and validated before use in a real environment.
