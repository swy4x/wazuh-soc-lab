# EICAR Test File Detection with Wazuh FIM

## Objective

Validate that Wazuh File Integrity Monitoring (FIM) detects the creation of a harmless EICAR antivirus test file.

## Lab setup

- Endpoint: Arch Linux
- Wazuh Agent: 4.14.5
- Wazuh Manager: 4.14.1
- Monitored directory: `/opt/wazuh-malware-lab`
- FIM mode: realtime

The monitored directory is configured with:

```xml
<directories realtime="yes">/opt/wazuh-malware-lab</directories>
```

## Test artifact

The EICAR test file was created at:

```
/opt/wazuh-malware-lab/eicar.com
```

The EICAR string is a standardized, harmless antivirus test string. It was **not executed**.

## Wazuh detection

Creating the file generated:

- Rule ID: **554**
- Level: **5**
- Description: `File added to the system.`
- FIM mode: realtime
- File size: **68 bytes**

Wazuh reported these hashes:

| Hash | Value |
|---|---|
| MD5 | `44d88612fea8a8f36de82e1278abb02f` |
| SHA-1 | `3395856ce81f2b7382dee72602f798b642f14140` |
| SHA-256 | `275a021bbfb6489e54d471899f7db9d1663fc695ec2fe2a2c4538aabf651fd0f` |

## Independent verification

The SHA-256 calculated locally matched the value reported by Wazuh.

The local `file` utility identified the artifact as an EICAR virus test file.

## SOC takeaway

This demonstrates the first stage of an endpoint artifact investigation:

1. A file appears on an endpoint.
2. FIM detects the creation.
3. Wazuh records file metadata and cryptographic hashes.
4. The SHA-256 can be used as an IOC for further investigation.

EICAR is intentionally harmless and is used here only to validate the detection pipeline.

## Evidence

### 📸 Screenshot — EICAR FIM detection
Capture the Wazuh Dashboard showing Rule 554, Level 5, for the EICAR file creation event.

### 📸 Screenshot — local hash verification
Capture the terminal showing the SHA-256, file size, and `file` output for the EICAR artifact.
