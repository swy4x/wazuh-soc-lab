# SSH Brute-Force Detection with Wazuh

## Lab objective

Detect repeated SSH authentication failures from the same source IP and correlate them into a higher-severity Wazuh alert.

All testing was performed against systems and networks controlled for this lab.

## Lab flow

```
SSH login attempt
      ↓
systemd journald
      ↓
Wazuh Agent (Arch Linux)
      ↓
Wazuh Manager
      ↓
SSH decoder
      ↓
Rule 5760
      ↓
Custom correlation rule 100003
      ↓
Wazuh Indexer
      ↓
Wazuh Dashboard
```

## Host log collection

The Arch Linux Wazuh Agent was configured to collect the systemd journal:

```xml
<localfile>
  <log_format>journald</log_format>
  <location>journald</location>
</localfile>
```

The agent forwards the authentication events to the local Wazuh Manager over TCP.

## Built-in rules observed

| Rule | Level | Purpose |
|---|---:|---|
| 5710 | 5 | SSH login attempt using a non-existent user |
| 5760 | 5 | SSH authentication failure |
| 5763 | 10 | Built-in SSH brute-force correlation |
| 2502 | 10 | Repeated password failures |
| 5503 | 5 | PAM login failure |
| 5557 | 5 | Password check failure |

The important parent rule for the custom correlation was **5760**, because it represents an individual SSH authentication failure.

## Custom correlation rule

The lab created rule `100003`:

```xml
<group name="ssh,bruteforce,local,">

  <rule id="100003" level="10" frequency="3" timeframe="60" ignore="60">
    <if_matched_sid>5760</if_matched_sid>
    <same_source_ip/>
    <description>LAB: SSH Brute Force - 3 authentication failures from the same source IP within 60 seconds</description>
    <mitre>
      <id>T1110</id>
    </mitre>
    <group>authentication_failed,brute_force,ssh,lab,mitre_t1110,</group>
  </rule>

</group>
```

### How the rule works

- `if_matched_sid=5760` — counts SSH authentication failures.
- `frequency=3` — requires three matching events.
- `timeframe=60` — the three events must occur within 60 seconds.
- `same_source_ip` — requires the failures to originate from the same IP.
- `ignore=60` — suppresses repeated alerts from this rule for 60 seconds after it fires.
- `T1110` — maps the detection to MITRE ATT&CK Brute Force.

## Validation

Before deployment, the Wazuh analysis engine was tested with:

```
/var/ossec/bin/wazuh-analysisd -t
```

The configuration validated successfully with no errors.

## Controlled test

Three failed SSH password attempts were generated against the lab Arch Linux host.

The final Wazuh Dashboard showed:

- Rule ID: **100003**
- Level: **10**
- Description: **LAB: SSH Brute Force - 3 authentication failures from the same source IP within 60 seconds**

The event was observed alongside the underlying `5760`, `5557`, `5503`, and `2502` events, demonstrating the difference between individual authentication events and the custom correlation alert.

## Screenshots

Add the lab screenshots manually under a future `screenshots/` directory:

```text
screenshots/
├── ssh-bruteforce-event-chain.png
├── ssh-bruteforce-rule-validation.png
└── ssh-bruteforce-detection.png
```

Suggested evidence:

1. **Event chain** — Wazuh Dashboard showing the related SSH events.
2. **Rule validation** — terminal showing rule `100003` and successful validation.
3. **Final detection** — Dashboard showing rule `100003`, Level 10.

## Key learning

This lab demonstrates that Wazuh detection is not limited to single log matches. Individual SSH failures can be used as the input to a correlation rule that detects repeated behavior and raises a higher-severity SOC alert.
