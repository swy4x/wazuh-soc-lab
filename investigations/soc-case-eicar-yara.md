# SOC Case: EICAR YARA Detection

## Alert

- Wazuh Rule: **100500**
- Severity: **Level 12**
- Agent: **001 (archlinux)**
- Detection source: `/var/ossec/logs/yara-results.log`
- Alert time: **2026-10-03 09:42:00.291 +0000**
- File: `/opt/wazuh-malware-lab/eicar.com`
- YARA rule: **Wazuh_EICAR_Test**

Alert message:

```text
YARA_MATCH rule=Wazuh_EICAR_Test path=/opt/wazuh-malware-lab/eicar.com
```

## Initial Observation

Wazuh reported a Level 12 YARA detection associated with a file monitored by File Integrity Monitoring. Because the alert was high severity, the correct response was to investigate the file and validate the detection rather than immediately delete or block it.

## Investigation

Manual YARA execution confirmed the detection:

```bash
sudo yara /opt/yara-rules/wazuh-malware-lab.yar /opt/wazuh-malware-lab/eicar.com
```

Result:

```text
Wazuh_EICAR_Test /opt/wazuh-malware-lab/eicar.com
```

The file was then inspected directly. The EICAR test string was present:

```text
X5O!P%@AP[4\\PZX54(P^)7CC)7}$EICAR-STANDARD-ANTIVIRUS-TEST-FILE!$H+H*
```

The file also contained additional lab markers added during integration testing. Those markers were not the YARA detection indicator.

Current file metadata at investigation time:

- Size: **213 bytes**
- SHA-256: `1d871c758d5c4b6f9e3c33eb7e46c2367c54a9326750e8dbd59ddb9dd4fd0dab`
- Path: `/opt/wazuh-malware-lab/eicar.com`

The current SHA-256 differs from the original 68-byte EICAR artifact because the file was modified during lab testing. Content-based YARA detection still matched because the recognizable EICAR string remained present.

## Evidence Correlation

The investigation correlated the following evidence:

```text
File created/modified
        ↓
Wazuh FIM
        ↓
YARA Active Response
        ↓
YARA match
        ↓
/var/ossec/logs/yara-results.log
        ↓
Wazuh Rule 100500
        ↓
Dashboard alert
```

The YARA results log confirmed the match, while the Wazuh alert supplied the alert timestamp. The YARA log itself did not contain a timestamp, so an exact YARA execution time was not inferred.

## Classification

**Benign security test artifact**

The file was intentionally used as part of the Wazuh FIM/YARA integration lab and contains the standardized EICAR antivirus test string. The detection itself was correct; the alert was not a false positive. The correct SOC conclusion is that a benign test artifact was successfully detected.

## Analyst Conclusion

The Wazuh alert detected a YARA match in `/opt/wazuh-malware-lab/eicar.com`. Manual YARA execution confirmed that the file matched the `Wazuh_EICAR_Test` rule, and file inspection confirmed the presence of the EICAR antivirus test string. The file was classified as a **benign security test artifact** because it was intentionally used for the lab YARA/FIM integration testing and no malicious execution was observed.

## Recommended Action

- No containment or malware-removal action was required for this controlled lab artifact.
- Record the investigation and evidence.
- Preserve the detection chain as a validated SOC lab scenario.
- In a real environment, continue with execution/process telemetry and endpoint context before deciding on containment.

## SOC Lesson

A high-severity alert is a starting point, not the final verdict.

The analyst should follow:

```text
Alert
 ↓
Validate detection
 ↓
Collect evidence
 ↓
Investigate artifact
 ↓
Correlate context
 ↓
Classify
 ↓
Respond
 ↓
Document
```
