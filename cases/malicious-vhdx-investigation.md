# Malicious VHDX Artifact Investigation

## 1. Case Summary

| Field | Details |
|---|---|
| Case type | Suspicious archive / suspected information stealer |
| Initial delivery | ZIP archive presented as a PDF-related document |
| Verdict | **High-confidence malicious artifact** |
| Analysis method | Static analysis on Arch Linux; public Triage sandbox report for an exact-hash executable match |
| Execution on analyst workstation | Not performed |
| Persistence | Not confirmed |

## 2. Incident Overview

A suspicious archive named `UTR_0040_102026_1487.zip` was received from a suspected scammer. The archive contained a virtual hard disk image rather than a conventional PDF document. The VHDX was inspected offline and mounted read-only with execution disabled.

The disk image contained an executable, two DLLs, and an unidentified binary file. The executable's SHA-256 matched a sample in a public Triage sandbox report. That report recorded behavior consistent with spyware/stealer activity, system discovery, and execution of dropped components.

## 3. File Inventory

| Artifact | Observed details |
|---|---|
| ZIP | `UTR_0040_102026_1487.zip` |
| VHDX | `UTR_0040_102026.F9FC35.vhdx` |
| Main executable | `UTR_0040_102026.exe` |
| DLL | `rIMe.dLL` |
| DLL | `EsdSip.dll` |
| Unknown file | `5702c641-23f5-4371-b786-28dea8dfe63.stl` |

The VHDX contained a small NTFS partition. The image was mounted read-only; the suspicious executable and DLLs were not launched.

## 4. SHA-256 Indicators

| Artifact | SHA-256 | Assessment |
|---|---|---|
| ZIP archive | `f10cea23c15de344a72a20c22ec3287d777ad3a1d2c3182f43d445c07690b848` | Suspicious delivery archive |
| Main EXE | `c74f55247aa2c72001024bd977c869cb5adce6f8604a4d32329f015b73ae2a49` | Exact-hash match to sandbox-analyzed sample |
| `rIMe.dLL` | `d8c33a1a7477a5ccb0cb869a3ce0cbf388c84e4fcf3354a43b28e4a1feeba834` | Suspicious DLL; no exact-hash public match established in this investigation |
| `.stl` file | `9a7e0911a09b7bca199976195cfb6252dd10cb4d8fc63837461d77d1214c7053` | Unidentified high-entropy binary |

## 5. Static Analysis Findings

### Main executable

- Identified as a 32-bit Windows PE executable.
- Contains the internal name `KafanInputService.exe` and references to Rime input-method APIs.
- Contains command-execution, registry, process, and shell-related imports and strings.
- Includes references to `cmd.exe`, a named pipe, mutexes, and application-data paths.
- These static findings alone do not establish maliciousness; the exact-hash sandbox match provides stronger evidence.

### DLL analysis

The DLL `rIMe.dLL` imports APIs associated with process-memory access, Windows hooks, service control, registry manipulation, shell execution, and networking. These imports show potential capabilities, not proof that every capability was used.

A large, nonstandard executable section with high entropy was also observed. This is an anomaly worth documenting, but it does not independently identify a specific packer or prove encryption.

### Unknown `.stl` file

The file had high measured entropy (approximately 7.94 bits per byte), no obvious common file signature, and no readable configuration recovered during the checks performed. Its purpose remains unknown. It is not classified as a payload without further evidence.

## 6. Public Sandbox Correlation

**Triage report:** https://tria.ge/261006-ytxrpaet5y/behavioral1

The report contained an executable named `N4915100511.exe` with the same SHA-256 as the locally extracted main executable:

`c74f55247aa2c72001024bd977c869cb5adce6f8604a4d32329f015b73ae2a49`

The report recorded the following behaviors:

- Enumerating connected drives and physical storage devices.
- Looking up the external IP address through `api.ipify.org`.
- Querying `ip-api.com` for geolocation information.
- Reading hostname, domain, system language, and configured country information.
- Enumerating video-capture drivers.
- Reading browser/profile data; the sandbox categorized this behavior under browser credential-access techniques.
- Executing a dropped executable and loading dropped DLLs.
- Executing a `cmd.exe` one-liner.
- Suspicious process enumeration, privilege adjustment, Windows-hook activity, and cross-process memory writes.

These are observations reported by the sandbox for the exact-hash sample. They are not claims that the same actions occurred on the analyst's own machine.

## 7. Network Indicators

The saved Triage report records the following flows for sandbox process PID 4816 / process ID 90:

| Domain / destination | Flow | Protocol / port | Bytes received by process | Bytes sent by process | Observation |
|---|---:|---|---:|---:|---|
| `khltt1.club` → `202.61.85.151` | 27 | TCP/6802 | 443,223 | 17,656 | Largest suspicious-domain flow; application protocol and payload not established |
| `khltt1.club` → `202.61.85.151` | 28 | TCP/6801 | 955 | 10,413 | Additional connection; purpose not established |
| `api.ipify.org` → `104.26.12.205:443` | 30 | TLS/HTTPS | 3,611 | 1,213 | GET `https://api.ipify.org/`; HTTP 200 observed, consistent with external-IP lookup |
| `ip-api.com` → `208.95.112.1:80` | 34 | TCP/HTTP | 914 | 692 | GET `/json/199.91.100.152?lang=zh-CN`; HTTP 200 observed, consistent with IP geolocation |
| `c.pki.goog` → `173.194.41.94:80` | 32 | TCP/HTTP | 5,555 | 1,140 | Certificate revocation-list requests with `Microsoft-CryptoAPI/10.0` user agent; likely certificate-validation context |

The domain `khltt1.club` resolved to `202.61.85.151` through DNS. The report's embedded Recorded Future SOAR feed gave `202.61.85.151` a risk score of 10 with the reason “Historical Spam Source”; the same feed gave `khltt1.club` a score of 0. These are source-specific reputation snapshots, not universal verdicts; a score of zero does not establish that a domain is safe.

The IP address `199.91.100.152` appeared as a parameter in the geolocation request. It should not be treated as attacker infrastructure without separate evidence. The `208.95.112.1` address is the observed DNS answer for `ip-api.com` and should not be called C2 based solely on its appearance in a third-party list.

**Network assessment:** The communication with `khltt1.club` on TCP ports 6801 and 6802 is the highest-priority network lead in the available records. Flow byte counts do not reveal payload contents and do not, by themselves, prove data exfiltration or command-and-control. The protocol and purpose of these two connections remain unconfirmed.

Other contextual indicators:
- `api.ipify.org`: external-IP lookup.
- `ip-api.com`: geolocation lookup.
- `https://www.baidu.com/s?wd=`: embedded search URL string in the executable; not confirmed as a network connection or C2.

## 8. Selected MITRE ATT&CK Mapping

| Technique | Name | Evidence |
|---|---|---|
| T1059.003 | Windows Command Shell | Sandbox observed a `cmd.exe` one-liner |
| T1012 | Query Registry | Country, language, and registry queries |
| T1016 | System Network Configuration Discovery | Hostname/domain queries |
| T1016.001 | Internet Connection Discovery | External-IP lookup services |
| T1082 | System Information Discovery | System and storage discovery |
| T1120 | Peripheral Device Discovery | Drive/device and capture-driver enumeration |
| T1125 | Video Capture | Capture-driver discovery; actual recording was not established by this finding alone |
| T1005 | Data from Local System | Browser/profile-data access was reported under collection-related analysis |
| T1552.001 | Credentials In Files | Sandbox classification associated with browser/profile-data access |
| T1555.003 | Credentials from Web Browsers | Sandbox classification associated with browser/profile-data access |
| T1614.001 | System Language Discovery | System language registry queries |

These mappings reflect sandbox-reported behavior and classifications. They do not prove successful credential theft or exfiltration.

## 9. Persistence Assessment

No definitive persistence mechanism was confirmed from the evidence collected. Service-control APIs were imported by a DLL, but API imports alone do not establish service installation or persistence. Registry strings found during static analysis did not establish a Run key, scheduled task, or confirmed autostart mechanism.

**Assessment:** Persistence not confirmed.

## 10. SOC Assessment

The exact SHA-256 match between the locally extracted executable and the public sandbox sample, combined with the sandbox's observed discovery, browser-data access, dropped-component execution, process-memory manipulation, and hook activity, supports a high-confidence malicious classification.

The archive's apparent document-related naming should not be trusted as evidence of its actual file type. The delivered object was a ZIP containing a VHDX, which in turn contained Windows executables and DLLs.

### Final disposition

**Verdict: Malicious — suspected spyware/stealer behavior.**

The investigation did not establish successful credential theft, data exfiltration, a specific command-and-control server, or persistence. Those should remain unconfirmed unless further evidence is obtained.

## 11. Recommended Response

1. Do not execute the EXE or DLLs on a normal workstation.
2. If the archive was opened only in a read-only analysis environment and the executable was never run, record that fact; do not assume the host is infected solely because the archive was inspected.
3. If the executable was run on a Windows device, isolate that device from networks and begin endpoint investigation. Review process execution, browser credential stores, persistence locations, and outbound connections.
4. Preserve the original archive, extracted artifacts, hashes, sandbox report, and timestamps as evidence.
5. Treat the domains as contextual indicators. Validate current ownership and legitimate usage before broad blocking; do not block certificate infrastructure solely because it appeared in the report.
6. Do not label the unknown `.stl` file as malware or configuration without additional evidence.

## 12. Evidence Limitations

- The local executable was analyzed statically and was not executed by the analyst.
- Dynamic findings came from a public sandbox report for a sample with an exact matching executable SHA-256.
- Exact-hash public matches were not established for the ZIP, VHDX, DLL, or `.stl` file in this investigation.
- The complete process tree and all dropped-file hashes were not finalized.
- Successful credential theft, exfiltration, persistence, and C2 activity remain unconfirmed.

## 13. References

- Public Triage behavioral report: https://tria.ge/261006-ytxrpaet5y/behavioral1
- MITRE ATT&CK technique references linked in the table above.

---

*Prepared as a defensive malware-analysis case study. No suspicious executable or DLL was run during local analysis.*
