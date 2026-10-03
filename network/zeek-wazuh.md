# Zeek → Wazuh Network Monitoring Lab

## Objective

Validate that Zeek network telemetry can be collected and investigated through Wazuh.

Zeek and Suricata are kept conceptually separate:

- **Zeek** produces rich protocol and connection telemetry.
- **Suricata** produces IDS signatures and alerts.
- **Wazuh** centralizes the telemetry and applies SIEM rules/correlation.

## Tested telemetry

The lab validated:

- Synthetic SSL/TLS telemetry
- DNS reconnaissance telemetry
- DNS reconnaissance escalation/correlation

Observed Wazuh detections included:

| Rule | Level | Purpose |
|---|---:|---|
| 100102 | 3 | Synthetic Zeek SSL event |
| 100304 | 7 | Zeek DNS reconnaissance |
| 100305 | 10 | Higher-severity DNS reconnaissance correlation |

## Important ingestion detail

A field-mapping conflict occurred because Zeek's native `id` field conflicted with the ingestion schema.

The working pipeline renamed the Zeek value to:

`data.zeek_id`

This allowed the Zeek event to be indexed without colliding with the existing document structure.

## Practical workflow

```
Network activity
      ↓
    Zeek
      ↓
Zeek structured logs
      ↓
Ingestion / Filebeat pipeline
      ↓
Wazuh Manager
      ↓
Custom Wazuh rules
      ↓
Wazuh Dashboard
```

## Evidence checklist

### 📸 Screenshot — Zeek SSL detection
Capture the Wazuh Dashboard event showing the validated synthetic SSL event and rule `100102`.

### 📸 Screenshot — DNS reconnaissance
Capture the Dashboard showing rule `100304` for the DNS reconnaissance event.

### 📸 Screenshot — DNS escalation
Capture the final higher-severity rule `100305` event.

## SOC takeaway

Zeek is valuable because it provides structured network-security telemetry that can be investigated beyond a single IDS signature. Wazuh can then turn that telemetry into centralized detections and correlations.
