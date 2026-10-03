# YARA Practical File-Hunting Lab

## Objective

Learn and validate YARA as a file-hunting engine before connecting its results to Wazuh.

**YARA is both:**

- a rule language used to describe file characteristics and detection logic;
- a command-line engine that evaluates those rules against files.

## Environment

- Arch Linux
- YARA 4.5.6
- Lab directory: `/opt/wazuh-malware-lab`
- Rule directory: `/opt/yara-rules`
- Test artifact: harmless EICAR antivirus test file

## Practical concepts completed

### 1. Exact string matching

A rule matched:

`EICAR-STANDARD-ANTIVIRUS-TEST-FILE`

This demonstrated basic content-based detection.

### 2. `any of them`

Multiple strings were defined and the rule matched when at least one was present.

### 3. `all of them`

The rule required every defined string to be present. A controlled test with two independent indicators produced no match when only one was present.

### 4. Boolean logic

The lab validated:

- `$one and $two`
- `$one or $two`
- `$one and not $two`

A negative test confirmed that adding the excluded indicator prevents an AND-NOT rule from matching.

### 5. File size

The built-in `filesize` value was used to detect files below a size threshold.

### 6. Compound conditions

A rule combined a content indicator with a file-size condition:

`$eicar and filesize < 100KB`

### 7. Hash matching

The YARA `hash` module was used to match the exact EICAR SHA-256.

The original file matched the known hash. A modified copy did not.

### 8. Hash versus content detection

Adding one harmless byte changed the SHA-256 from:

`275a021bbfb6489e54d471899f7db9d1663fc695ec2fe2a2c4538aabf651fd0f`

to:

`d01fa970d3e882abcd3df0673d2cb58a381205958512cb9d6e724e949fab23ac`

The hash rule stopped matching, while the EICAR string rule still matched.

This demonstrates the difference between:

- **hash IOC detection** — exact file identity;
- **content/rule detection** — characteristics that can survive small file changes.

### 9. Regex

A regex rule detected a harmless private IPv4 test pattern:

`192.168.1.[0-9]{1,3}`

### 10. Metadata

The `meta` section was used to record rule author, description, and severity.

Metadata describes the rule; it does not decide whether the rule matches.

### 11. Private helper rules

A `private` helper rule was referenced by a main rule. The helper matched internally but was not printed as a standalone result.

### 12. Rule tags

Tags were added to a rule to categorize it for rule-set management.

### 13. ELF module

The `elf` module was used to identify Linux executables and scan `/usr/bin`.

### 14. PE module

The `pe` module identified a harmless Wine `cmd.exe` PE sample.

A PE architecture test also validated a 32-bit I386 executable.

### 15. PE + indicator logic

A combined PE/string rule was tested against a plain text file. It produced no match because the file contained the indicator but was not a PE.

### 16. Recursive EICAR hunting

Recursive YARA scanning was used against the lab directory and detected both the original and modified EICAR artifacts through content-based matching.

## YARA → Wazuh integration

The practical phase now includes a verified automated pipeline:

```
File created/modified
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
/var/ossec/logs/yara-results.log
  ↓
Wazuh Manager
  ↓
Rule 100500 / Level 12
  ↓
Dashboard
```

The integration uses:

- YARA 4.5.6
- Wazuh Agent 4.14.5
- Wazuh Manager 4.14.1
- Active Response command `yara-scan`
- YARA rule `Wazuh_EICAR_Test`
- Wazuh rule `100500`

The final Dashboard event showed:

```text
YARA_MATCH rule=Wazuh_EICAR_Test path=/opt/wazuh-malware-lab/eicar.com
```

with Wazuh Rule 100500 at Level 12.

## Important limitation / learning boundary

The practical toolbox and integration are complete. The next phase is **formal YARA-language learning**, rather than more copy-paste integration work.

Planned language lessons:

1. Rule structure
2. Identifiers
3. String types
4. String modifiers
5. Conditions
6. Boolean expressions
7. Modules
8. Rule reuse and helper rules
9. Writing a detection rule from scratch
10. Positive and negative testing

## Evidence checklist

Screenshots are intentionally collected during the learning/documentation phase.

### 📸 Basic YARA match
Terminal showing the EICAR string rule matching the test artifact.

### 📸 Hash detection
Terminal showing the exact EICAR SHA-256 match.

### 📸 Hash change
Original and modified SHA-256 values and the failed hash-rule match for the modified file.

### 📸 String survives modification
Modified EICAR detected by the content rule.

### 📸 ELF detection
ELF rule matching Linux executables.

### 📸 PE detection
PE rule matching Wine `cmd.exe`.

### 📸 Final YARA → Wazuh alert
Dashboard showing Rule 100500, Level 12, and the `YARA_MATCH` full log.

## Status

**Completed.** The standalone YARA practical toolbox and the automated YARA → Wazuh integration have both been validated end-to-end.
