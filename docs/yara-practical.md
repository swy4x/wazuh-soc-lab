# YARA Practical Lab

## Purpose

This document records the practical YARA experiments completed before the formal YARA-language lessons.

The practical phase was intentionally kept separate from the language-learning phase.

## Environment

- YARA: 4.5.6
- Endpoint: Arch Linux
- Lab directory: `/opt/wazuh-malware-lab`

## What was tested

### 1. String matching

A rule matched the EICAR test string inside the harmless EICAR artifact.

### 2. Multiple strings

Tested:

- `any of them`
- `all of them`

This demonstrated that YARA conditions can combine multiple string identifiers.

### 3. Boolean logic

Tested:

- AND
- OR
- NOT

This demonstrated that detection conditions can be composed rather than relying on a single string.

### 4. File size

A condition was used to test the size of the EICAR artifact.

### 5. SHA-256

YARA's hash functionality was used to match the original EICAR SHA-256.

A modified copy kept the EICAR text but had a different SHA-256. The result demonstrated:

- content/string matching can survive a file modification;
- exact hash matching cannot.

### 6. Regex

A regular-expression rule matched a controlled private IPv4 pattern.

### 7. Metadata

Rule metadata was added for analyst context, including author, description, and severity.

Metadata does not itself perform the detection.

### 8. Private helper rules

A private helper rule was referenced by a public rule.

The helper did not appear as the final match; the public rule was reported.

### 9. Tags

Rule tags were tested to attach classification labels to a rule.

### 10. ELF module

The ELF module was used to identify executable/shared-object binaries.

### 11. PE module

The PE module was used to identify a Windows PE file present in the Wine environment.

PE architecture matching was also tested.

### 12. PE + indicator logic

A PE condition was combined with a controlled string indicator.

A plain text file containing the indicator did not satisfy the PE condition, demonstrating that both conditions matter.

### 13. Recursive hunting

Recursive YARA scanning was used to hunt the lab directory for the EICAR indicator.

## Important lesson

YARA can detect different kinds of evidence:

```
Exact content
    +
Patterns
    +
File properties
    +
Hashes
    +
Executable structure
```

The detection method should match the investigation goal.

## Practical phase status

The practical YARA toolbox is complete.

The next learning phase is **formal YARA language**, starting from the structure of a rule and then learning each part one at a time.

This document should not be treated as a replacement for those lessons.
