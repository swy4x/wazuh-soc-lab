# YARA Practical File-Hunting Lab

## Objective

Learn and validate YARA as a standalone file-hunting engine before integrating YARA results into Wazuh.

**YARA is both:**

- a rule language used to describe file characteristics and detection logic;
- a command-line engine that evaluates those rules against files.

This section documents the practical experiments completed so far. The formal YARA-language lessons are kept separate and will follow after the practical toolbox is complete.

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

A negative test confirmed that adding the excluded indicator prevents an `AND NOT` rule from matching.

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

## Important limitation

The current YARA lab is **standalone**.

```
File
  ↓
YARA
  ↓
Terminal match
```

YARA detections are not yet automatically sent to the Wazuh Dashboard.

A future integration will build:

```
File
  ↓
YARA
  ↓
Detection result
  ↓
Wazuh Agent / log ingestion
  ↓
Wazuh Manager rule
  ↓
Dashboard alert
```

## Evidence checklist

### 📸 Screenshot — basic YARA match
Capture the terminal showing `EICAR_Test_File` matching `eicar.com`.

### 📸 Screenshot — clean recursive scan
Capture the terminal showing the recursive scan where only the intended EICAR artifact is reported.

### 📸 Screenshot — hash detection
Capture the terminal showing `EICAR_Hash_Test` matching the original file.

### 📸 Screenshot — hash change
Capture the original and modified SHA-256 values and the absence of a hash-rule match for the modified file.

### 📸 Screenshot — string survives modification
Capture the modified EICAR file being detected by the string rule.

### 📸 Screenshot — ELF detection
Capture the ELF rule matching Linux executables.

### 📸 Screenshot — PE detection
Capture the PE rule matching Wine `cmd.exe`.

### 📸 Screenshot — recursive hunting
Capture a short section of the recursive EICAR scan showing both EICAR artifacts.

## Next phase

After the practical toolbox is complete, learn the YARA language explicitly:

1. Rule structure
2. Identifiers
3. String types
4. Modifiers
5. Conditions
6. Boolean expressions
7. Modules
8. Rule reuse and helper rules
9. Writing a detection rule from scratch
10. Positive and negative testing
