# SSH Invalid-User Detection

## Objective

Verify that an SSH login attempt using a non-existent username is collected by Wazuh through systemd journald.

## Test

A controlled login attempt used:

```text
ssh wronguser@<LAB_HOST_IP>
```

## Observed event

| Field | Value |
|---|---|
| Agent | 001 / archlinux |
| Source IP | 10.41.169.183 |
| Source user | wronguser |
| Decoder | sshd |
| Location | journald |
| Rule | 5710 |
| Level | 5 |
| Description | sshd: Attempt to login using a non-existent user |

The observed event was:

```
Failed password for invalid user wronguser from 10.41.169.183
```

## Detection flow

```
SSH client
  ↓
sshd
  ↓
systemd journal
  ↓
Wazuh Agent
  ↓
Wazuh Manager
  ↓
Rule 5710
  ↓
Indexer / Dashboard
```

## SOC lesson

The important information is not only that authentication failed. The event also provides the source IP, attempted username, endpoint, decoder, and rule identity.

The event can later become an input to correlation logic such as SSH brute-force detection.

## Position in the SSH Detection Chain

Rule 5710 is a built-in Wazuh rule. It is not the custom brute-force rule.

The lab's detection layers are:

```
SSH invalid user
      ↓
Wazuh 5710
      ↓
SSH authentication failure
      ↓
Wazuh 5760
      ↓
three failures from same source within 60 seconds
      ↓
Custom Wazuh 100003
```

This separation is important: Rule 5710 provides evidence about the attempted account, while Rule 100003 represents the higher-level brute-force behavior.
