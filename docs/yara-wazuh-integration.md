# YARA → Wazuh Integration — Complete Lab Record

> **Goal:** when a monitored file changes, Wazuh should automatically launch YARA, capture the result, and bring that result back into Wazuh as a normal alert.

This was validated end-to-end using the harmless EICAR antivirus test artifact.

---

# 1.  Objective

Build this chain:

```
File created / modified
        ↓
Wazuh FIM
        ↓
Rule 554 / 550
        ↓
Active Response
        ↓
yara-scan
        ↓
YARA
        ↓
YARA_MATCH
        ↓
Wazuh log collection
        ↓
Manager Rule 100500
        ↓
Dashboard
```

This is the final working architecture.

---

# 2.  Why connect FIM and YARA?

FIM and YARA answer different questions.

### FIM

> **What changed?**

### YARA

> **Does the changed file match my detection logic?**

### Wazuh

> **How do we correlate, alert, store, and investigate the result?**

The integration therefore creates a chain rather than turning one tool into another.

---

# 3.  YARA Rule

Location:

```
/opt/yara-rules/wazuh-malware-lab.yar
```

Rule:

```yara
rule Wazuh_EICAR_Test
{
    meta:
        author = "Swayam"
        description = "Detects the harmless EICAR antivirus test string"
        severity = "high"

    strings:
        $eicar = "EICAR-STANDARD-ANTIVIRUS-TEST-FILE"

    condition:
        $eicar
}
```

### Important design choice

The rule is content-based.

It does not depend on the original EICAR SHA-256.

That means the integration demonstrates actual YARA matching rather than simply checking whether a file has one hard-coded hash.

---

# 4.  FIM Trigger

The monitored directory:

```
/opt/wazuh-malware-lab
```

uses realtime FIM:

```xml
<directories realtime="yes">/opt/wazuh-malware-lab</directories>
```

Relevant events:

| Rule | Meaning |
|---:|---|
| 554 | File added |
| 550 | File modified |

These are the Active Response triggers.

---

# 5.  Active Response Configuration

The command definition:

```xml
<command>
  <name>yara-scan</name>
  <executable>yara-scan</executable>
  <timeout_allowed>no</timeout_allowed>
</command>
```

The Active Response:

```xml
<active-response>
  <disabled>no</disabled>
  <command>yara-scan</command>
  <location>local</location>
  <rules_id>554,550</rules_id>
</active-response>
```

### Why only 554 and 550?

Because the configuration is intended the scanner to run when a file is:

- created;
- modified.

Deletion does not provide a file for YARA to scan.

---

# 6.  Scanner Scope

Active Response executable:

```
/var/ossec/active-response/bin/yara-scan
```

The scanner is restricted to:

```
/opt/wazuh-malware-lab/*
```

This is deliberate.

The lab automation is not intended to become an unrestricted endpoint scanner.

---

# 7.  Scanner Logic

The final scanner performs these steps:

```
Receive Wazuh event
       ↓
Read one JSON line
       ↓
Require command = add
       ↓
Extract FIM file path
       ↓
Check lab-directory restriction
       ↓
Check file exists
       ↓
Run YARA
       ↓
Extract first match
       ↓
Write YARA_MATCH
       ↓
Exit
```

The important working input handling is:

```bash
INPUT=""
IFS= read -r INPUT
```

---

# 8.  Debugging: The `cat` Problem

The first implementation used:

```bash
INPUT="$(cat)"
```

It looked reasonable at first.

But the Active Response process did not terminate.

### Why?

`cat` waits for EOF.

Wazuh keeps the stdin pipe available to the response process, so EOF did not arrive as expected.

The process was observed waiting on a pipe.

### Fix

Read one event:

```bash
IFS= read -r INPUT
```

After this change, the scanner processed the event and exited.

### SOC/engineering lesson

A security automation script must understand the **execution contract of the system launching it**.

The problem was not YARA.

The problem was stdin handling.

---

# 9.  Extracting the Event

The scanner uses `jq` to extract:

### Command

```
.command
```

### FIM path

```
.parameters.alert.data.syscheck.path
```

with the alternate path:

```
.parameters.alert.syscheck.path
```

The scanner therefore works from the Wazuh event instead of using a hard-coded filename.

---

# 10.  Running YARA

The scanner executes:

```
/usr/bin/yara /opt/yara-rules/wazuh-malware-lab.yar "$PATH_TO_SCAN"
```

If YARA finds a match, the scanner converts it into a predictable event:

```
YARA_MATCH rule=Wazuh_EICAR_Test path=/opt/wazuh-malware-lab/eicar.com
```

---

# 11.  Result Log

The normalized result is written to:

```
/var/ossec/logs/yara-results.log
```

Example:

```
YARA_MATCH rule=Wazuh_EICAR_Test path=/opt/wazuh-malware-lab/eicar.com
```

This is important because the YARA script itself is not the final alerting system.

It produces telemetry that Wazuh can consume.

---

# 12.  Agent Collection

The Agent collects the YARA result log:

```xml
<localfile>
  <log_format>syslog</log_format>
  <location>/var/ossec/logs/yara-results.log</location>
</localfile>
```

Therefore the workflow becomes:

```
YARA
 ↓
normalized log
 ↓
Wazuh Agent
 ↓
Wazuh Manager
```

---

# 13.  Manager Rule 100500

The Manager uses:

```xml
<group name="yara,local,">
  <rule id="100500" level="12">
    <match>YARA_MATCH</match>
    <description>YARA detected a malware-test indicator in a FIM-monitored file</description>
    <group>yara,malware_detection,file_integrity,lab,</group>
  </rule>
</group>
```

### What this rule does

It turns the normalized scanner output:

```
YARA_MATCH
```

into a Wazuh alert.

---

# 14.  Final Validation

The final test modified:

```
/opt/wazuh-malware-lab/eicar.com
```

by appending a harmless test marker.

This triggered FIM.

FIM triggered Active Response.

Active Response launched YARA.

YARA matched the EICAR content.

The result log contained:

```
YARA_MATCH rule=Wazuh_EICAR_Test path=/opt/wazuh-malware-lab/eicar.com
```

The scanner process was then checked and had completed rather than remaining stuck.

---

# 15.  Dashboard Validation

The resulting Dashboard alert contained:

| Field | Observed value |
|---|---|
| Index | `wazuh-alerts-4.x-2026.10.03` |
| Agent ID | `001` |
| Agent name | `archlinux` |
| Rule ID | `100500` |
| Rule level | `12` |
| Location | `/var/ossec/logs/yara-results.log` |
| Manager | `wazuh.manager` |
| Full log | `YARA_MATCH rule=Wazuh_EICAR_Test path=/opt/wazuh-malware-lab/eicar.com` |
| Timestamp | `2026-10-03T09:42:00.291+0000` |

This is the evidence that the complete chain worked.

---

# 16.  Configuration Debugging

There was another real configuration mistake during implementation.

An Active Response block was initially placed after:

```xml
</ossec_config>
```

That produced an invalid-root configuration error.

The incorrect block was removed.

The command and Active Response configuration were then placed in their valid locations.

Manager validation:

```
/var/ossec/bin/wazuh-analysisd -t
```

completed successfully.

The Manager restarted successfully afterward.

---

# 17.  Final End-to-End Proof

The entire chain was observed:

```
1. File modification
        ↓
2. FIM Rule 550
        ↓
3. Active Response
        ↓
4. yara-scan
        ↓
5. YARA match
        ↓
6. yara-results.log
        ↓
7. Wazuh Agent
        ↓
8. Manager Rule 100500
        ↓
9. Indexer
        ↓
10. Dashboard alert
```

This validates the integration end to end.

---

# 18.  What This Lab Actually Teaches

The primary engineering lesson is architectural.

A SOC rarely depends on one tool doing everything.

Instead:

```
FIM
→ tells us something changed

YARA
→ tells us whether the file matches detection logic

Wazuh
→ turns that evidence into a centralized alert

Dashboard
→ gives the analyst an investigation surface
```

That is the real value of the integration.

---

# 19.  Safety / Scope

- EICAR was used only as a harmless antivirus test artifact.
- The artifact was never executed.
- YARA Active Response is restricted to the lab directory.
- Network tests were performed only against controlled/authorized targets.
- This is a learning implementation, not a claim of production-ready malware response.

---

# 20.  Status

### Integration Status

Complete

### Verified:

- FIM trigger
- Active Response
- YARA execution
- YARA match
- normalized result logging
- Agent collection
- Manager Rule 100500
- Indexer storage
- Dashboard alert
- Active Response stdin debugging
- Wazuh configuration debugging

### Next:

**Formal YARA language learning from zero.**

## Exact Custom Rules Used by the Integration

### Endpoint YARA rule

```yara
rule Wazuh_EICAR_Test
{
    meta:
        author = "Swayam"
        description = "Detects the harmless EICAR antivirus test string"
        severity = "high"

    strings:
        $eicar = "EICAR-STANDARD-ANTIVIRUS-TEST-FILE"

    condition:
        $eicar
}
```

### Manager Wazuh Rule 100500

```xml
<group name="yara,local,">
  <rule id="100500" level="12">
    <match>YARA_MATCH</match>
    <description>YARA detected a malware-test indicator in a FIM-monitored file</description>
    <group>yara,malware_detection,file_integrity,lab,</group>
  </rule>
</group>
```

### FIM trigger rules

The Active Response is bound to the built-in FIM rules:

| Rule | Event |
|---:|---|
| 554 | File added |
| 550 | File modified |

### Final detection path

```
FIM 554/550
    ↓
Wazuh Active Response
    ↓
YARA Wazuh_EICAR_Test
    ↓
YARA_MATCH
    ↓
Wazuh Rule 100500
    ↓
Indexer
    ↓
Dashboard
```

The exact rule source is also maintained in **[Custom Detection Rules Reference](custom-rules.md)**.
