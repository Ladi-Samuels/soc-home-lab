# Day 5 — Round 2: PowerShell-Delivered Meterpreter Reverse Shell

## Objective

Continuation of Day 4. Change the delivery method from browser download to PowerShell `Invoke-WebRequest` and observe how Sysmon telemetry differs from Round 1.

## Environment

| Role | Host | IP |
|---|---|---|
| Attacker | Kali Linux | 192.168.65.3 |
| Victim | Windows 11 | 192.168.65.4 |

## What Changed From Round 1

The original plan was to deliver the payload via an SMB share hosted on Kali in the directory containing `evil.exe`. SMB was blocked by Defender, so the delivery method pivoted to **PowerShell `Invoke-WebRequest`** pulling directly from the Kali HTTP server.

## Attack Chain

1. Payload (`evil.exe`) hosted on Kali HTTP server, port 8080
2. Listener started on Kali, port 4444
3. **Windows 11 victim executed PowerShell `Invoke-WebRequest`** to download `evil.exe` from the Kali HTTP server
4. User executed `evil.exe` — reverse shell established back to Kali at **16:56:13**

## The Key Finding — Missing Telemetry

What made this round different from Day 4:

- ❌ **Event ID 15 absent**
- ❌ **No Mark of the Web (Zone.Identifier)**
- ❌ Windows had no record that the file came from the internet

PowerShell's `Invoke-WebRequest` **does not apply the Zone.Identifier alternate data stream** the way a browser does. The file lands on disk with:

- No internet origin warning
- No SmartScreen prompt
- No "are you sure you want to run this?" dialog

This is exactly why attackers favor PowerShell for payload delivery — it bypasses the MotW protection mechanism that browser-based delivery would otherwise trigger.

## Parent Process Comparison

The `ParentImage` field in Event ID 1 is now one of the most important telemetry signals:

| Parent → Child | Interpretation |
|---|---|
| `msedge.exe` → `suspicious.exe` | Browser-delivered |
| `explorer.exe` → `suspicious.exe` | User manually executed |
| `powershell.exe` → `suspicious.exe` | Script-delivered (Living off the Land) |

In Round 1, the parent was `msedge.exe`. In Round 2, the parent is `explorer.exe` (the user double-clicked the downloaded file). The PowerShell session that downloaded the file (`powershell.exe → evil.exe` via Invoke-WebRequest) is a separate signal — **Ingress Tool Transfer via Living off the Land**.

## Sysmon Event IDs — Round 2 vs Round 1

### Consistent Across Both Rounds

| Event ID | Meaning |
|---|---|
| 11 | File created |
| 1 | Process created |
| 13 | Registry value set |
| 3 | Network connection |
| 2 | Timestomping |
| 5 | Process terminated |

### Present in Round 1, Absent in Round 2

| Event ID | Meaning | Why It's Missing |
|---|---|---|
| 22 | DNS query | PowerShell connected directly to Kali IP, no DNS lookup |
| 15 | File stream / MotW | `Invoke-WebRequest` doesn't apply Zone.Identifier |
| 12 | Registry object created | Session terminated before attacker established persistence |

## MITRE ATT&CK Mapping

| Technique | Description |
|---|---|
| T1059 / T1059.001 | Command and Scripting Interpreter: PowerShell |
| T1105 | Ingress Tool Transfer (PowerShell downloading `evil.exe` from Kali) |
| T1204.002 | User Execution: Malicious File |

## Indicators of Compromise

| IOC Type | Value |
|---|---|
| Malicious file | `evil.exe` (same binary as Round 1) |
| MD5 | `82FC735C28439C7A4F694AFA793079B8` |
| SHA-256 | `1C80E26DD48F1B1CA4A849D5E6336245431F44B4FA4FE4583BBE37D09D48EB86` |
| C2 IP | `192.168.65.3` |
| C2 Port | `4444` (unchanged) |
| Delivery Port | `8080` (unchanged) |
| Delivery Method | PowerShell `Invoke-WebRequest` (**changed** from browser) |
| ParentImage | `explorer.exe` (**changed** from `msedge.exe`) |

## Why This Matters

PowerShell downloading an executable to `Downloads` with no Zone.Identifier = **higher risk than browser download** because the user gets no warning before execution.

## Confidence Assessment

**High.** If this were a real attack, Defender's inability to detect the payload would have given the attacker initial access, lateral movement potential, command and control, and exfiltration capability. The absence of Event ID 12 suggests the session terminated before the attacker could establish persistence — a longer dwell time would likely produce registry object creation as the attacker pivoted toward persistence mechanisms.
