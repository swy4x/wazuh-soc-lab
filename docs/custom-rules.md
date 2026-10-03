# Custom Detection Rules Reference

> This document records the custom detection logic used in the lab and keeps each engine's identifiers separate.

## 1. Wazuh Rule 100003 — SSH brute-force correlation

This rule was written for the SSH brute-force lab:

```xml
<group name="ssh,bruteforce,local,">

  <rule id="100003" level="10" frequency="3" timeframe="60" ignore="60">
    <if_matched_sid>5760</if_matched_sid>
    <same_source_ip/>
    <description>LAB: SSH Brute Force - 3 authentication failures from the same source IP within 60 seconds</description>
    <mitre>
      <id>T1110</id>
    </mitre>
    <group>authentication_failed,brute_force,ssh,lab,mitre_t1110,</group>
  </rule>

</group>
```

### Rule behavior

- **Input:** Wazuh Rule 5760, an SSH authentication failure.
- **Frequency:** 3 matching events.
- **Timeframe:** 60 seconds.
- **Correlation key:** source IP.
- **Ignore period:** 60 seconds after the custom rule fires.
- **MITRE mapping:** T1110, Brute Force.
- **Level:** 10.

The rule deliberately builds on the built-in Wazuh SSH detection instead of replacing it.

---

## 2. Wazuh Rule 100500 — YARA result alert

This rule converts the normalized output of the YARA Active Response scanner into a normal Wazuh alert:

```xml
<group name="yara,local,">
  <rule id="100500" level="12">
    <match>YARA_MATCH</match>
    <description>YARA detected a malware-test indicator in a FIM-monitored file</description>
    <group>yara,malware_detection,file_integrity,lab,</group>
  </rule>
</group>
```

The scanner writes:

```
YARA_MATCH rule=Wazuh_EICAR_Test path=/opt/wazuh-malware-lab/eicar.com
```

Rule 100500 then turns that line into the Dashboard alert observed in the final integration test.

---

## 3. YARA Rule — Wazuh_EICAR_Test

The final integration rule is:

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

This is a content-based rule. It intentionally does not depend on the original EICAR SHA-256, which allows the lab to demonstrate that modified copies can still be detected by recognizable content.

---

## 4. YARA Practical Rules

### Basic EICAR rule

```yara
rule EICAR_Test_File
{
    strings:
        $eicar = "EICAR-STANDARD-ANTIVIRUS-TEST-FILE"

    condition:
        $eicar
}
```

### Regex test

```yara
$ip = /192\.168\.1\.[0-9]{1,3}/
```

### ELF test

```yara
import "elf"

rule ELF_File_Test
{
    condition:
        elf.type == elf.ET_EXEC or elf.type == elf.ET_DYN
}
```

### PE test

```yara
import "pe"

rule PE_File_Test
{
    condition:
        pe.is_pe
}
```

### PE architecture test

```yara
import "pe"

rule PE_32bit_Test
{
    condition:
        pe.machine == pe.MACHINE_I386
}
```

### PE plus indicator test

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

The practical phase also tested `any of them`, `all of them`, AND, OR, NOT, file size, SHA-256, metadata, private helper rules, tags, and recursive scanning.

---

## 5. Suricata Custom Signature Catalogue

The lab used these custom Suricata SIDs:

| SID | Detection |
|---:|---|
| 1000001 | TCP SYN Scan / Port Sweep |
| 1000002 | ICMP Recon / Ping Sweep |
| 1000003 | SSH Connection Burst |
| 1000004 | SMB Scan |
| 1000005 | RDP Scan |
| 1000006 | HTTP Port Scan |
| 1000007 | HTTPS Port Scan |
| 1000008 | UDP Scan / Burst |
| 1000009 | DNS Query Burst |
| 1000010 | TCP FIN Scan |
| 1000011 | TCP NULL Scan |
| 1000012 | TCP Xmas Scan |

The controlled Nmap experiment validated SID 1000001 and showed it entering Wazuh under Rule 86601.

The current repository does not contain the original Suricata `local.rules` source file, so exact signature bodies are not reconstructed here.

---

## 6. Zeek/Wazuh Custom Rule Catalogue

| Rule | Level | Purpose |
|---:|---:|---|
| 100102 | 3 | Synthetic Zeek SSL event |
| 100304 | 7 | DNS reconnaissance |
| 100305 | 10 | DNS reconnaissance correlation |

The Zeek integration also required mapping Zeek's native `id` field to:

```
data.zeek_id
```

The current repository records the verified IDs and behavior, but does not contain the original Zeek/Wazuh rule-source blocks.

---

## 7. Identifier Separation

The lab deliberately uses different identifier namespaces:

```
Wazuh rule 100003
Wazuh rule 100500
       ≠
Suricata SID 1000001
       ≠
YARA rule Wazuh_EICAR_Test
```

This distinction matters when investigating alerts. The identifier tells the analyst which detection engine produced the original signal.

---

## 8. Safety and Provenance

These rules were created and tested for a controlled learning environment.

- SSH tests used authorized lab systems.
- Network signatures were tested against controlled traffic.
- EICAR was used as a harmless antivirus test artifact and was never executed.
- The YARA Active Response scanner is restricted to `/opt/wazuh-malware-lab/*`.
- The Suricata and Zeek catalogues are documented only to the level verified by the repository; missing original source is not invented.
