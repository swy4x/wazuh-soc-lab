# Zeek → Wazuh Network Monitoring

## Objective

Validate that Zeek telemetry can be ingested and investigated through Wazuh.

## Tool roles

- **Zeek** — structured network-security telemetry.
- **Suricata** — network IDS signatures.
- **Wazuh** — centralized collection, rules, correlation, and investigation.

## Tested detections

| Rule | Level | Purpose |
|---|---:|---|
| 100102 | 3 | Synthetic Zeek SSL event |
| 100304 | 7 | Zeek DNS reconnaissance |
| 100305 | 10 | DNS reconnaissance correlation |

## Data flow

```
Network activity
 ↓
Zeek
 ↓
Structured telemetry
 ↓
Ingestion / Filebeat pipeline
 ↓
Wazuh
 ↓
Custom rules
 ↓
Dashboard
```

## Field-mapping issue

Zeek's native `id` field conflicted with the Wazuh ingestion mapping.

The working pipeline maps that value to:

```
data.zeek_id
```

This allowed the event to be indexed correctly.

## SOC lesson

Zeek is useful when the analyst needs structured network context rather than only an IDS signature.

## Custom Wazuh Rule Catalogue

The Zeek integration used these custom Wazuh rules:

| Rule | Level | Purpose |
|---:|---:|---|
| 100102 | 3 | Synthetic Zeek SSL event |
| 100304 | 7 | DNS reconnaissance |
| 100305 | 10 | DNS reconnaissance correlation |

The integration also required mapping Zeek's native `id` field to `data.zeek_id` so it would not conflict with Wazuh's reserved `id` mapping.

### DNS detection relationship

```
Zeek DNS telemetry
      ↓
Rule 100304
      ↓
Related or repeated DNS activity
      ↓
Rule 100305
```

The repository preserves the verified IDs, levels, telemetry purpose, and field-mapping fix. The original Zeek/Wazuh rule-source blocks are not currently stored, so their exact syntax is not reconstructed.
