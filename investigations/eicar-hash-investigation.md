# EICAR Hash Investigation

## Objective

Investigate the SHA-256 produced by a Wazuh FIM alert and correlate the artifact with an external malware-intelligence source.

## Artifact

Path:

```
/opt/wazuh-malware-lab/eicar.com
```

SHA-256:

```
275a021bbfb6489e54d471899f7db9d1663fc695ec2fe2a2c4538aabf651fd0f
```

Size:

```
68 bytes
```

File type:

```
EICAR virus test files
```

## VirusTotal lookup

The exact SHA-256 was searched in VirusTotal.

Observed result:

- Detection: **65/67**
- Popular threat label: `virus.eicar/test`
- Family labels included: `eicar`, `test`, `file`
- File size: **68 B**
- The hash matched the artifact exactly.

The vendor detections predominantly identified the artifact as an **EICAR test file**, with several vendors explicitly indicating that it is not a real virus.

## Analyst conclusion

The high detection count does **not** mean this lab artifact is real malware.

EICAR is a standardized antivirus test artifact intentionally designed to trigger security products. The correct SOC conclusion is:

> The file is the known EICAR antivirus test artifact. Wazuh successfully detected its creation and produced a reliable SHA-256 IOC. The VirusTotal result confirms the identity of the test artifact rather than indicating a genuine malware infection.

## Investigation workflow learned

```
File appears
    ↓
Wazuh FIM alert
    ↓
Extract SHA-256
    ↓
Verify hash locally
    ↓
Search IOC in VirusTotal
    ↓
Correlate detection names and file identity
    ↓
Make an evidence-based analyst conclusion
```

## Important safety note

The EICAR file was **not executed**. It was used only as a harmless detection test.

## Evidence

- Wazuh Dashboard: Rule 554 file creation alert.
- VirusTotal hash lookup: exact SHA-256 match.
