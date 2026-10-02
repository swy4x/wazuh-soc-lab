# SSH Invalid-User Authentication Detection

## Objective

Verify that an SSH authentication failure on the Arch Linux host is collected through systemd journal by the Wazuh agent and reaches the Wazuh Dashboard.

## Test

A controlled SSH login attempt used a non-existent username from the lab client:

```text
ssh wronguser@<LAB_HOST_IP>
```

The failed authentication generated SSH events in the Arch Linux journal.

## Observed Wazuh event

| Field | Value |
|---|---|
| Agent ID | `001` |
| Agent name | `archlinux` |
| Agent IP | `127.0.0.1` |
| Source IP | `10.41.169.183` |
| Source user | `wronguser` |
| Decoder | `sshd` |
| Location | `journald` |
| Wazuh rule | `5710` |
| Wazuh level | `5` |
| Rule description | `sshd: Attempt to login using a non-existent user` |
| MITRE technique | `T1110.001 Password Guessing`, `T1021.004 SSH` |

## Example event

```text
Failed password for invalid user wronguser from 10.41.169.183
```

## Investigation

The event establishes:

- `10.41.169.183` = SSH source/client
- `wronguser` = non-existent account used in the attempt
- `archlinux` = monitored endpoint
- Wazuh received the event through `journald`
- The `sshd` decoder extracted the source IP and username
- Wazuh classified the event with rule `5710`

## Detection pipeline

```text
SSH client
   ↓
Arch Linux sshd
   ↓
systemd journal
   ↓
Wazuh Agent 4.14.5
   ↓
Wazuh Manager 4.14.1
   ↓
Wazuh rule 5710
   ↓
Wazuh Indexer
   ↓
Wazuh Dashboard
```

## Next step

Use repeated controlled authentication failures from the same source to test the custom SSH brute-force correlation rule `100002`.
