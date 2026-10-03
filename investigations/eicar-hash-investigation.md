# EICAR Hash Investigation

## Objective

Use the SHA-256 from a Wazuh FIM event to identify the known test artifact and understand how an analyst should interpret external reputation data.

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

## VirusTotal observation

The exact SHA-256 was searched in VirusTotal.

Observed:

- 65/67 detections
- EICAR/test-related labels
- exact hash match
- 68 B file size

Multiple vendors identified the artifact as an EICAR test file rather than a genuine malware sample.

## Analyst interpretation

A high detection count by itself is not enough to conclude that an endpoint is infected.

In this case, the artifact is the standardized EICAR antivirus test file. The evidence supports:

```
Known EICAR test artifact
        +
Successful Wazuh FIM detection
        +
Reliable SHA-256 IOC
```

## Investigation workflow

```
FIM alert
 ↓
Extract SHA-256
 ↓
Verify locally
 ↓
Search IOC
 ↓
Compare file identity / labels
 ↓
Document the conclusion
```

The file was not executed.
