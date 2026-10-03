# YARA → Wazuh Integration

## Objective

Connect YARA to Wazuh so that a file change detected by Wazuh FIM automatically triggers a YARA scan and returns the result to Wazuh as a normal alert.

## Final workflow

```
File created / modified
        ↓
Wazuh FIM
        ↓
Rule 554 / Rule 550
        ↓
Active Response
        ↓
yara-scan
        ↓
YARA
        ↓
YARA_MATCH
        ↓
/var/ossec/logs/yara-results.log
        ↓
Wazuh Agent
        ↓
Manager Rule 100500
        ↓
Dashboard
```

## YARA rule

Location:

```
/opt/yara-rules/wazuh-malware-lab.yar
```

Rule used:

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

The rule identifies the EICAR content string. It does not depend on the original file hash.

## FIM trigger

The lab directory is monitored in realtime:

```xml
<directories realtime="yes">/opt/wazuh-malware-lab</directories>
```

Relevant FIM events:

- Rule 554 — file added
- Rule 550 — file modified

These events are used as the trigger for Active Response.

## Active Response command

The Manager defines:

```xml
<command>
  <name>yara-scan</name>
  <executable>yara-scan</executable>
  <timeout_allowed>no</timeout_allowed>
</command>
```

Active Response:

```xml
<active-response>
  <disabled>no</disabled>
  <command>yara-scan</command>
  <location>local</location>
  <rules_id>554,550</rules_id>
</active-response>
```

## Agent-side scanner

Location:

```
/var/ossec/active-response/bin/yara-scan
```

The scanner:

1. reads one JSON event line;
2. requires the `add` command;
3. extracts the FIM file path;
4. accepts only files under `/opt/wazuh-malware-lab/`;
5. verifies the file exists;
6. runs YARA;
7. extracts the match;
8. writes a normalized result;
9. exits.

The result format is:

```
YARA_MATCH rule=Wazuh_EICAR_Test path=/opt/wazuh-malware-lab/eicar.com
```

## Important debugging fix

The first implementation used:

```bash
INPUT="$(cat)"
```

That waited for EOF and left the Active Response process hanging.

The working approach reads one event line:

```bash
IFS= read -r INPUT
```

This allows the scanner to process the event and exit.

## Wazuh result log

The scanner writes to:

```
/var/ossec/logs/yara-results.log
```

The Wazuh Agent collects that file as a local syslog-formatted log source.

## Final Wazuh rule

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

## Validation

The integration was tested by modifying the EICAR artifact in the monitored directory.

The result log produced:

```
YARA_MATCH rule=Wazuh_EICAR_Test path=/opt/wazuh-malware-lab/eicar.com
```

The Wazuh Dashboard then showed:

| Field | Observed value |
|---|---|
| Rule | 100500 |
| Level | 12 |
| Agent | 001 / archlinux |
| Location | `/var/ossec/logs/yara-results.log` |
| Full log | `YARA_MATCH rule=Wazuh_EICAR_Test path=/opt/wazuh-malware-lab/eicar.com` |

Observed timestamp: **October 3, 2026 @ 09:42:00.291**.

## What was actually proven

This lab proved the complete automation chain:

1. FIM detects a file event.
2. Wazuh starts Active Response.
3. The endpoint invokes YARA.
4. YARA evaluates the file.
5. The result is written to a Wazuh-collected log.
6. Wazuh detects the YARA result.
7. The Dashboard displays the resulting alert.

## Scope

- EICAR was used only as a harmless test artifact.
- The file was not executed.
- The scanner is restricted to the lab directory.
- Testing is limited to systems controlled for the lab.
