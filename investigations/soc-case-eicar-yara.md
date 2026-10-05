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


# Threat Hunt: FIM File Creation and Modification

## Hunt Hypothesis

> A suspicious file may be created or modified on the endpoint.

The hunt used Wazuh File Integrity Monitoring (FIM) telemetry rather than starting from a high-severity malware alert.

## File Creation — Rule 554

A controlled test file was created:

```bash
echo "SOC-HUNT-TEST" | sudo tee /opt/wazuh-malware-lab/soc-hunt-test.txt
```

Wazuh generated Rule **554**:

- Rule: **554**
- Level: **5**
- Event: **added**
- Detection mode: **realtime**
- Path: `/opt/wazuh-malware-lab/soc-hunt-test.txt`
- Size: **14 bytes**
- Owner: **root:root**
- Permissions: **rw-r--r--**
- SHA-256: `154395f8c336773df0a07f37f947cd5a47d0fdaf57e07fed2d179d73be163f40`
- Alert timestamp: **2026-10-05 09:24:50.428 +0000**

The file contained the controlled lab marker `SOC-HUNT-TEST` and was classified as benign in the lab context.

## File Modification — Rule 550

The same file was deliberately modified:

```bash
echo "MODIFIED" | sudo tee -a /opt/wazuh-malware-lab/soc-hunt-test.txt
```

Wazuh generated Rule **550**:

- Rule: **550**
- Level: **7**
- Event: **modified**
- Detection mode: **realtime**
- Path: `/opt/wazuh-malware-lab/soc-hunt-test.txt`
- Size: **14 → 23 bytes**
- Changed attributes: **size, mtime, md5, sha1, sha256**
- SHA-256 before: `154395f8c336773df0a07f37f947cd5a47d0fdaf57e07fed2d179d73be163f40`
- SHA-256 after: `a0d462755e2927015d22a339a64655d64bcd7f561f8ef9f58c830d5fcfd17adb`
- Alert timestamp: **2026-10-05 09:27:49.443 +0000**

Final file content:

```text
SOC-HUNT-TEST
MODIFIED
```

The file remained owned by `root:root` with permissions `rw-r--r--`.

## Analyst Assessment

The FIM alerts were valid detections of file creation and modification. They were **benign controlled lab events** because the analyst intentionally created and modified the file during the hunt.

The presence of Rule 550 and its MITRE mapping to T1565.001 does not by itself prove malicious data manipulation. Alert severity and technique mapping provide context; the analyst must correlate the event with file content, ownership, timing, process context, and the surrounding investigation.

## Threat-Hunting Lesson

A SOC analyst can hunt for low-level telemetry that has not crossed a higher alert threshold. The demonstrated workflow was:

```text
Hypothesis
 ↓
Search telemetry
 ↓
Generate controlled event
 ↓
Validate FIM detection
 ↓
Inspect artifact
 ↓
Compare before/after evidence
 ↓
Classify activity
```

This complements alert-driven investigation by helping identify activity and potential detection gaps that may not produce high-severity alerts.
