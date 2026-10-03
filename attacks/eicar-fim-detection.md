# EICAR Detection with Wazuh FIM

## Objective

Validate that Wazuh File Integrity Monitoring detects creation of the harmless EICAR antivirus test artifact.

## Environment

- Endpoint: Arch Linux
- Wazuh Agent: 4.14.5
- Wazuh Manager: 4.14.1
- Monitored directory: `/opt/wazuh-malware-lab`
- FIM mode: realtime

Configuration:

```xml
<directories realtime="yes">/opt/wazuh-malware-lab</directories>
```

## Artifact

```
/opt/wazuh-malware-lab/eicar.com
```

The EICAR string is a standardized antivirus test artifact. It was not executed.

## Observed Wazuh event

- Rule: **554**
- Level: **5**
- Description: `File added to the system.`
- File size: **68 bytes**

Hashes reported by Wazuh:

| Hash | Value |
|---|---|
| MD5 | `44d88612fea8a8f36de82e1278abb02f` |
| SHA-1 | `3395856ce81f2b7382dee72602f798b642f14140` |
| SHA-256 | `275a021bbfb6489e54d471899f7db9d1663fc695ec2fe2a2c4538aabf651fd0f` |

The local SHA-256 calculation matched the Wazuh value.

## Detection flow

```
EICAR file created
 ↓
FIM / syscheckd
 ↓
Rule 554
 ↓
Wazuh Agent
 ↓
Wazuh Manager
 ↓
Dashboard
```

## SOC lesson

FIM provides the first observable fact:

> A monitored file appeared.

The file metadata and hashes can then be used for further investigation.
