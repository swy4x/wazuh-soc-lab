# SSH Brute-Force Detection

## Objective

Detect repeated SSH authentication failures from the same source IP and raise a higher-severity Wazuh alert.

All testing was performed against controlled lab systems.

## Detection flow

```
SSH authentication failure
        ↓
systemd journald
        ↓
Wazuh Agent
        ↓
Wazuh Manager
        ↓
Rule 5760
        ↓
Custom Rule 100003
        ↓
Indexer / Dashboard
```

## Relevant built-in rules

| Rule | Level | Purpose |
|---|---:|---|
| 5710 | 5 | Invalid SSH user |
| 5760 | 5 | SSH authentication failure |
| 5763 | 10 | Built-in SSH brute-force correlation |
| 2502 | 10 | Repeated password failures |
| 5503 | 5 | PAM login failure |
| 5557 | 5 | Password check failure |

## Custom rule

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

## Rule logic

- `if_matched_sid 5760` — use SSH authentication failures as the input.
- `frequency 3` — require three matching events.
- `timeframe 60` — those events must occur within 60 seconds.
- `same_source_ip` — correlate failures from the same source.
- `ignore 60` — suppress repeated alerts from this rule for 60 seconds after firing.
- `T1110` — maps the behavior to MITRE ATT&CK Brute Force.

## Validation

The Wazuh analysis engine was validated before deployment:

```
/var/ossec/bin/wazuh-analysisd -t
```

Three controlled failed SSH password attempts produced the custom Rule 100003 alert.

## SOC lesson

A single authentication failure is an event.

Repeated failures from the same source within a defined time window become a **behavioral detection**.

That distinction is the main learning point of this lab.


## Complete Custom Rule Source

The exact custom rule used in the lab is:

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

### Evaluation sequence

```
5760 failure #1
       ↓
5760 failure #2
       ↓
5760 failure #3
       ↓
same source IP
       ↓
within 60 seconds
       ↓
Rule 100003
```

The 60-second ignore setting suppresses repeated firing of this custom rule for the configured period after it triggers. It does not remove the underlying authentication events.