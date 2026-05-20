# Day 6 — Round 3: C2 Port Blending (8443)

## Objective

Continuation of Days 4 and 5. Change only the C2 port from `4444` to `8443` and observe whether port-based detection alone would catch the attack, or whether behavioral signatures remain the only reliable signal.

## Environment

| Role | Host | IP |
|---|---|---|
| Attacker | Kali Linux | 192.168.65.3 |
| Victim | Windows 11 | 192.168.65.4 |

## What Changed From Round 2

Only the listener port. Delivery method, parent process, and binary characteristics remained the same as Round 2.

| Variable | Round 2 | Round 3 |
|---|---|---|
| Delivery | PowerShell `Invoke-WebRequest` | PowerShell `Invoke-WebRequest` |
| C2 Port | 4444 | **8443** |
| Parent Process | explorer.exe | explorer.exe |
| Payload | evil.exe | evil_r3.exe |

## Operational Note

Defender continued to automatically re-enable itself, requiring **Tamper Protection to be disabled** to maintain the attack window. Disabling Tamper Protection is itself a significant indicator of attacker presence and would be a high-priority detection in a real SOC.

## Attack Chain

1. Payload `evil_r3.exe` generated on Kali, configured to call back on port **8443**
2. Hosted on Kali HTTP server, port 8080
3. Listener started on Kali, port **8443**
4. Windows 11 victim executed PowerShell `Invoke-WebRequest` to download `evil_r3.exe`
5. User executed `evil_r3.exe` — reverse shell established back to Kali at **16:04:26**

## Why Port 8443 Matters

Port `8443` is commonly used for legitimate HTTPS alternate traffic, web applications, internal APIs, and management consoles. This outbound traffic **blends into normal business traffic** and is far less likely to:

- Trigger a firewall alert
- Be flagged by an analyst on review
- Stand out in flow logs

Since the port itself cannot be blocked without breaking legitimate services, detection must focus on the **behavior on the port**, not the port itself.

## MITRE ATT&CK Mapping

| Technique | Description |
|---|---|
| T1571 | Non-Standard Port |
| T1059.001 | Command and Scripting Interpreter: PowerShell |
| T1105 | Ingress Tool Transfer |
| T1204.002 | User Execution: Malicious File |

## Comparison Across All Three Rounds

| | Round 1 | Round 2 | Round 3 |
|---|---|---|---|
| Delivery | Browser (msedge.exe) | PowerShell IWR | PowerShell IWR |
| Parent process | msedge.exe | explorer.exe | explorer.exe |
| C2 Port | 4444 (flagged) | 4444 (same) | **8443 (blending)** |
| MotW present | ✅ Yes | ❌ No | ❌ No |
| Event ID 22 (DNS) | ✅ Yes | ❌ No | ❌ No |
| Event ID 15 (MotW) | ✅ Yes | ❌ No | ❌ No |
| Event ID 12 (Registry obj) | ✅ Yes | ❌ No | ❌ No |

## Sysmon Event IDs — Consistent Across All Three Rounds

These are the **reliable detection anchors** regardless of delivery method or port:

| Event ID | Meaning |
|---|---|
| 11 | File created |
| 1 | Process created |
| 3 | Network connection |
| 2 | Timestomping |
| 5 | Process terminated |

## Indicators of Compromise

| IOC Type | Value |
|---|---|
| Malicious file | `evil_r3.exe` |
| MD5 | `07FE559BCF6B97FB290A00C3CE71220A` |
| SHA-256 | `590474B36144EC9A219D56DDC714094FFECDAC6F586CCC924B290698B9FF7320` |
| IMPHASH | `858F2D67A04EAC8E503A0B0B92530EE2` |
| C2 IP | `192.168.65.3` |
| C2 Port | `8443` (**changed**) |
| Delivery Port | `8080` (unchanged) |
| Delivery Method | PowerShell `Invoke-WebRequest` |
| ParentImage | `explorer.exe` |

## Core Conclusion

**Port-based detection alone is insufficient.** The reliable detection signal across all three rounds was not the port — it was the **behavioral chain**:

> Browser or PowerShell delivers an executable → an unsigned binary with no metadata executes → process initiates outbound TCP connection → process terminates abruptly

This chain persists regardless of:
- Delivery method (browser, PowerShell, theoretically SMB)
- Destination port (4444, 8443, or any other)
- Parent process (msedge.exe, explorer.exe)

It forms the foundation for the custom detection rules being developed in **Phase 2 — Wazuh Integration**.

## Confidence Assessment

**High.** Across three rounds, Defender's inability to consistently detect the payload would have given a real attacker initial access, lateral movement potential, command and control, and exfiltration capability. The combination of MotW absence + non-standard port + Living-off-the-Land delivery is a textbook stealth chain that demands behavioral detection rather than signature- or port-based filtering.
