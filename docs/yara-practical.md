# YARA Practical Lab — Complete Experiment Record

> **This document records what we actually tested before beginning the formal YARA-language course.**

The practical phase was deliberately separated from the formal language lessons.

That means this document answers:

> **“What did we make YARA do?”**

The future language course answers:

> **“How does the YARA language work, and how can I write rules myself?”**

---

# 1. Environment

| Item | Value |
|---|---|
| OS | Arch Linux |
| YARA | 4.5.6 |
| Lab directory | `/opt/wazuh-malware-lab` |

---

# 2.  What YARA is

YARA is both a **rule language** and a **detection engine**.

A useful mental model:

```
YARA rule
   ↓
written in YARA language
   ↓
yara executable
   ↓
file/data evaluation
   ↓
match / no match
```

The `.yar` file is the rule source.

The `yara` command is the engine that evaluates it.

---

# 3.  Experiment 1 — Basic String Matching

The first rule:

```yara
rule EICAR_Test_File
{
    strings:
        $eicar = "EICAR-STANDARD-ANTIVIRUS-TEST-FILE"

    condition:
        $eicar
}
```

The rule was tested against:

```
/opt/wazuh-malware-lab/eicar.com
```

Result:

```
EICAR_Test_File /opt/wazuh-malware-lab/eicar.com
```

### What we learned

YARA can detect a file based on **content**.

The rule did not need to know the filename.

---

# 4.  Experiment 2 — Recursive Scanning

Command tested:

```
yara -r /opt/wazuh-malware-lab/eicar.yar /opt/wazuh-malware-lab
```

An interesting result appeared: the rule file itself could match because the EICAR indicator literally appeared inside the rule source.

### Lesson

Recursive scanning searches files in the target tree.

The rule source is not automatically excluded from the scan target.

This is a small but important practical detail.

---

# 5.  Experiment 3 — `any of them`

Multiple strings were defined and the condition used:

```
any of them
```

### Meaning

At least one referenced string must match.

Mental model:

```
A OR B OR C
```

This is useful when several indicators can independently identify the same artifact.

---

# 6.  Experiment 4 — `all of them`

The same style of rule was tested using:

```
all of them
```

### Meaning

Every referenced string must match.

Mental model:

```
A AND B AND C
```

This is useful when a rule should require several pieces of evidence instead of one.

---

# 7.  Experiment 5 — Boolean Logic

Separate tests were performed using:

- AND
- OR
- NOT

Conceptually:

```
A AND B → both must be true
A OR B  → at least one must be true
NOT A   → A must be false
```

### Lesson

YARA conditions can become logical detection expressions rather than simple string searches.

---

# 8.  Experiment 6 — File Size

A file-size condition was tested against the EICAR artifact.

The EICAR file was known to be:

```
68 bytes
```

### Lesson

YARA can use **file properties** as detection evidence.

It does not have to depend only on text.

---

# 9. #⃣ Experiment 7 — SHA-256 Matching

YARA hash functionality was tested using:

```
hash.sha256(0, filesize)
```

The original EICAR file matched its expected SHA-256:

```
275a021bbfb6489e54d471899f7db9d1663fc695ec2fe2a2c4538aabf651fd0f
```

---

# 10.  Experiment 8 — Hash vs Content

A modified EICAR copy was created.

### Original

```
275a021bbfb6489e54d471899f7db9d1663fc695ec2fe2a2c4538aabf651fd0f
```

### Modified

```
d01fa970d3e882abcd3df0673d2cb58a381205958512cb9d6e724e949fab23ac
```

The hash rule stopped matching.

The content/string rule still matched.

### This was one of the most important experiments.

```
Exact hash
   ↓
Exact identity
   ↓
Small content change can break it

Content indicator
   ↓
Pattern evidence
   ↓
Can survive a modification
```

So:

> A hash is excellent for identifying an exact known file, while a content/pattern rule can identify related or modified variants.

---

# 11.  Experiment 9 — Regular Expression

A controlled test file contained a private IPv4 pattern.

The rule used:

```yara
$ip = /192\.168\.1\.[0-9]{1,3}/
```

The pattern matched the test data.

### Lesson

YARA can search for **patterns**, not only exact strings.

---

# 12.  Experiment 10 — Metadata

Metadata was added:

```yara
meta:
    author = "Swayam"
    description = "Detects a private IPv4 test pattern"
    severity = "low"
```

The rule still matched.

### Important distinction

Metadata describes the rule.

It does **not** perform the detection.

---

# 13.  Experiment 11 — Private Helper Rules

A private helper rule was created and referenced by a public rule.

Only the public rule appeared as the reported detection.

### Lesson

Private rules can act as reusable internal logic without becoming separate final results.

---

# 14.  Experiment 12 — Rule Tags

A rule was tested with tags:

```yara
rule Network_Test : network lab
```

The rule matched normally.

### Lesson

Tags classify a rule and can help organize a larger ruleset.

---

# 15.  Experiment 13 — ELF Module

The ELF module was used to inspect Linux binaries.

Rule:

```yara
import "elf"

rule ELF_File_Test
{
    condition:
        elf.type == elf.ET_EXEC or elf.type == elf.ET_DYN
}
```

Recursive testing against system binaries produced many matches.

Examples included system/application binaries such as:

- bash
- Wireshark
- containerd
- Hyprland
- node
- Suricata
- Zeek

### Lesson

YARA can inspect executable structure rather than simply searching raw text.

---

# 16.  Experiment 14 — PE Module

The PE module was tested against:

```
/usr/lib/wine/i386-windows/cmd.exe
```

Rule:

```yara
import "pe"

rule PE_File_Test
{
    condition:
        pe.is_pe
}
```

The file matched.

### Lesson

The PE module lets YARA reason about Windows Portable Executable structure.

---

# 17.  Experiment 15 — PE Architecture

The rule:

```yara
import "pe"

rule PE_32bit_Test
{
    condition:
        pe.machine == pe.MACHINE_I386
}
```

matched the Wine `cmd.exe` file.

### Lesson

YARA can use properties inside executable formats, not only generic file content.

---

# 18.  Experiment 16 — PE Import Test

the lab tested:

```
pe.imports("KERNEL32.dll", "CreateProcessA")
```

The test produced **no output**.

### Correct conclusion

The tested file did not satisfy that exact import condition.

We deliberately did **not** invent another API name simply to force a match.

### Lesson

In detection engineering:

> **No match is a valid result.**

A test should be recorded honestly rather than manipulated until it succeeds.

---

# 19.  Experiment 17 — PE + Indicator Logic

A rule combined:

```
pe.is_pe
```

with the indicator:

```
"WAZUH-MALWARE-LAB"
```

A plain text file containing the indicator did not match.

Why?

Because:

```
indicator = true
PE condition = false
```

Therefore the combined condition was false.

### Lesson

AND-style conditions really constrain a detection.

---

# 20.  Experiment 18 — Recursive EICAR Hunting

The EICAR rule was used recursively against:

```
/opt/wazuh-malware-lab
```

The scan found:

```
EICAR_Test_File /opt/wazuh-malware-lab/eicar-modified.com
EICAR_Test_File /opt/wazuh-malware-lab/eicar.com
```

This was especially useful because the modified EICAR file had a different SHA-256 but still contained the recognizable EICAR content.

---

# 21.  Experiment 19 — Compiled Rules

A compiled-rule experiment was attempted.

The command used an unsupported:

```
-o
```

option and failed with:

```
unknown option '-o'
```

No successful compiled-rule artifact was claimed.

### Why this remains documented

The repository records **real experiments**, including failed ones.

The point is not to make every experiment look successful.

Compiled rules were not required for the current SOC learning path, so this topic was intentionally left unfinished.

---

# 22.  What the Practical Phase Proved

The experiments progressed from simple evidence to richer evidence:

```
literal string
     ↓
multiple indicators
     ↓
Boolean logic
     ↓
file properties
     ↓
hash identity
     ↓
regex patterns
     ↓
metadata / organization
     ↓
ELF structure
     ↓
PE structure
     ↓
PE-specific properties
```

The deeper lesson:

> **The detection method should match the investigation question.**

Examples:

| Question | Useful evidence |
|---|---|
| Is this exact known file? | Hash |
| Does this file contain an indicator? | String |
| Does it resemble a pattern? | Regex |
| Is it a Linux executable? | ELF |
| Is it a Windows executable? | PE |
| Does it satisfy several conditions? | Boolean logic |

---

# 23.  Relationship to Wazuh

The practical YARA work eventually became the YARA → Wazuh integration.

```
FIM detects change
       ↓
Active Response
       ↓
YARA evaluates file
       ↓
normalized YARA result
       ↓
Wazuh alert
```

The complete integration is documented separately:

 [YARA → Wazuh Integration](yara-wazuh-integration.md)

---

# 24.  Practical Phase Status

### Completed

- String matching
- Multiple-string logic
- Boolean conditions
- File-size checks
- SHA-256 matching
- Hash/content comparison
- Regex
- Metadata
- Private helper rules
- Tags
- ELF module
- PE module
- PE architecture
- PE + indicator logic
- Recursive hunting
- YARA → Wazuh integration

### Not claimed as completed

- Compiled-rule workflow

---

# 25.  Next: Formal YARA Language

The next stage starts **from zero**.

The intended learning progression is:

```
What is a YARA rule?
        ↓
rule structure
        ↓
rule name
        ↓
strings section
        ↓
string identifiers
        ↓
condition section
        ↓
Boolean operators
        ↓
modifiers
        ↓
modules
        ↓
write rules from scratch
        ↓
debug rules
        ↓
SOC-quality detection logic
```

This document is the practical experiment record.

Formal language instruction is maintained separately and will build from syntax fundamentals to detection development.


# 26. Exact Rules Used During the Practical Phase

The practical phase included concrete rule definitions.

## EICAR string rule

```yara
rule EICAR_Test_File
{
    strings:
        $eicar = "EICAR-STANDARD-ANTIVIRUS-TEST-FILE"

    condition:
        $eicar
}
```

## Regex rule

```yara
$ip = /192\\.168\\.1\\.[0-9]{1,3}/
```

## ELF rule

```yara
import "elf"

rule ELF_File_Test
{
    condition:
        elf.type == elf.ET_EXEC or elf.type == elf.ET_DYN
}
```

## PE rule

```yara
import "pe"

rule PE_File_Test
{
    condition:
        pe.is_pe
}
```

## PE architecture rule

```yara
import "pe"

rule PE_32bit_Test
{
    condition:
        pe.machine == pe.MACHINE_I386
}
```

## PE plus indicator rule

```yara
import "pe"

rule PE_String_Test
{
    strings:
        $indicator = "WAZUH-MALWARE-LAB"

    condition:
        pe.is_pe and $indicator
}
```

## Final integration rule

```yara
rule Wazuh_EICAR_Test
{
    meta:
        author = "Swayam"
        description = "Detects the harmless EICAR antivirus test string"
        severity = "high"

    strings:
        $eicar = "EICAR-STANDARD-ANTIVIRUS-TEST-FILE"

    condition:
        $eicar
}
```

Location:

```
/opt/yara-rules/wazuh-malware-lab.yar
```

The complete verified custom-rule catalogue is also maintained in the custom-rules reference document.