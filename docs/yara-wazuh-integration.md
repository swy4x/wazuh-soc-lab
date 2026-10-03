# YARA → Wazuh Integration Lab

## Objective

Build and validate an end-to-end malware-test detection pipeline in which Wazuh File Integrity Monitoring (FIM) detects a file event, Wazuh Active Response invokes a local YARA scanner, and the YARA result is ingested back into Wazuh as a Level 12 alert.

This lab uses the harmless EICAR antivirus test artifact. The file was never executed.

## Environment

- Host OS: Arch Linux
- Wazuh Manager: 4.14.1
- Wazuh Agent: 4.14.5-1
- Wazuh deployment: Docker-based single-node stack
- Agent ID: 001
- Agent name: `archlinux`
- Wazuh Dashboard: HTTPS on the local Docker deployment
- YARA: 4.5.6
- FIM monitored directory: `/opt/wazuh-malware-lab`
- YARA rules: `/opt/yara-rules/wazuh-malware-lab.yar`
- YARA result log: `/var/ossec/logs/yara-results.log`
- Active Response script: `yara-scan`

## Tools used

### Wazuh

**Wazuh Agent**
- Collects endpoint telemetry.
- Runs FIM/syscheckd.
- Executes the local Active Response script.

**Wazuh Manager**
- Receives agent events.
- Applies rules.
- Starts Active Response for selected FIM events.
- Ingests the YARA result log.

**Wazuh Dashboard**
- Used to verify the final alert and investigate fields.

**Wazuh FIM / syscheckd**
- Watches `/opt/wazuh-malware-lab` in realtime.
- Generates Rule 554 for file creation and Rule 550 for file modification.

**Wazuh Active Response**
- Uses the `yara-scan` command when Rule 554 or Rule 550 fires.

### YARA

YARA 4.5.6 is used as the file-hunting engine.

The detection rule currently used is:

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

The rule detects the EICAR content string rather than relying on the file's exact hash.

## Integration architecture

```
File created/modified
        ↓
Wazuh FIM / syscheckd
        ↓
Rule 554 or Rule 550
        ↓
Wazuh Manager Active Response
        ↓
Agent-side yara-scan script
        ↓
YARA 4.5.6
        ↓
YARA_MATCH result
        ↓
/var/ossec/logs/yara-results.log
        ↓
Wazuh Agent log collection
        ↓
Wazuh Manager Rule 100500
        ↓
Wazuh Dashboard Level 12 alert
```

## FIM configuration

The malware lab directory is monitored in realtime:

```xml
<directories realtime="yes">/opt/wazuh-malware-lab</directories>
```

The agent configuration was validated with:

```text
sudo /var/ossec/bin/wazuh-agentd -t
```

## Active Response configuration

The manager contains a command definition:

```xml
<command>
  <name>yara-scan</name>
  <executable>yara-scan</executable>
  <timeout_allowed>no</timeout_allowed>
</command>
```

The Active Response is enabled for FIM file-add and file-modification events:

```xml
<active-response>
  <disabled>no</disabled>
  <command>yara-scan</command>
  <location>local</location>
  <rules_id>554,550</rules_id>
</active-response>
```

The manager configuration was validated with:

```text
wazuh-analysisd -t
```

The manager was then restarted successfully.

## Agent-side YARA scanner

The Active Response receives a JSON command on stdin. An initial implementation incorrectly used `cat`, which waited for EOF because Active Response provides a single JSON line.

The final implementation reads exactly one line:

```bash
INPUT=""
IFS= read -r INPUT
```

It then:

1. Ignores empty input.
2. Requires `command == add`.
3. Extracts the FIM path from:
   - `parameters.alert.data.syscheck.path`
   - or `parameters.alert.syscheck.path`
4. Restricts scanning to:
   `/opt/wazuh-malware-lab/*`
5. Verifies the path is a regular file.
6. Runs YARA against the file.
7. Extracts the first YARA match.
8. Writes a normalized result to `/var/ossec/logs/yara-results.log`.
9. Exits cleanly.

The result format is:

```text
YARA_MATCH rule=Wazuh_EICAR_Test path=/opt/wazuh-malware-lab/eicar.com
```

## Wazuh detection rule

The manager uses a local rule to convert the YARA result into a Wazuh alert:

```xml
<group name="yara,local,">
  <rule id="100500" level="12">
    <match>YARA_MATCH</match>
    <description>YARA detected a malware-test indicator in a FIM-monitored file</description>
    <group>yara,malware_detection,file_integrity,lab,</group>
  </rule>
</group>
```

The manager configuration was validated successfully.

## End-to-end validation

The EICAR artifact was scanned and the YARA result log contained:

```text
===== EICAR FILE =====
-rw-r--r-- 1 swayam swayam 69 ... /opt/wazuh-malware-lab/eicar.com

YARA_MATCH rule=Wazuh_EICAR_Test path=/opt/wazuh-malware-lab/eicar.com
```

The Wazuh Dashboard then produced the final alert:

- Rule ID: **100500**
- Level: **12**
- Description: `YARA detected a malware-test indicator in a FIM-monitored file`
- Agent: **001 / archlinux**
- Location: `/var/ossec/logs/yara-results.log`
- Full log:
  `YARA_MATCH rule=Wazuh_EICAR_Test path=/opt/wazuh-malware-lab/eicar.com`
- Groups: `yara, malware_detection, file_integrity, lab`

The observed Dashboard timestamp was **Oct 3, 2026 @ 09:42:00.291**.

## What this proves

The integration is not just a standalone YARA test. It proves a complete SOC-style automated workflow:

1. Wazuh notices a file event.
2. FIM identifies the affected file.
3. Active Response receives the event.
4. The endpoint invokes YARA automatically.
5. YARA evaluates the file using a detection rule.
6. The result is written into a Wazuh-collected log.
7. Wazuh Manager detects the YARA result.
8. The Dashboard displays a high-severity alert for analyst investigation.

## Troubleshooting lesson

### Active Response stdin

The first scanner implementation waited indefinitely because it used:

```bash
INPUT="$(cat)"
```

Active Response does not require waiting for EOF in this workflow. Reading one JSON line with:

```bash
IFS= read -r INPUT
```

allowed the script to process the event and exit.

This is an important integration detail when writing Wazuh Active Response scripts.

## Scope and safety

- EICAR was used only as a harmless antivirus test artifact.
- The EICAR file was not executed.
- The YARA scanner is restricted to the lab directory.
- This lab is intended for systems and networks owned or controlled for testing.

## Evidence checklist

Screenshots are collected during the learning/documentation phase.

### 📸 FIM trigger
Dashboard showing the Rule 554 or Rule 550 event that initiated YARA scanning.

### 📸 YARA result
Terminal showing `YARA_MATCH` in `/var/ossec/logs/yara-results.log`.

### 📸 Final Wazuh alert
Dashboard showing Rule 100500, Level 12, and the `YARA_MATCH` full log.

## Status

**Completed and verified end-to-end.**

The next phase is formal YARA language learning: rule structure, identifiers, strings, modifiers, conditions, Boolean logic, modules, helper rules, and writing detection rules from scratch.
