# SOC Home Lab

A self-built security operations lab focused on detection engineering, attacker telemetry analysis, and SIEM integration. The lab simulates a small enterprise environment using virtualized attacker and victim hosts, and documents the full detection chain from attacker action to defender visibility.

## Lab Architecture

**Host platform:** Mac Mini M4 (Apple Silicon)
**Hypervisor:** UTM
**Network mode:** Shared network — `192.168.65.0/24`

| Host | Role | IP | OS |
|---|---|---|---|
| SOC-ATTACKER-01 | Attacker | 192.168.65.3 | Kali Linux |
| SOC-WINDOWS-02 | Victim | 192.168.65.4 | Windows 11 (ARM64) |

Additional hosts (Windows 11 #2, Metasploitable) are provisioned as needed for specific labs.

## Phases

### Phase 1 — Meterpreter Detection via Sysmon ✅ Complete

Three rounds of Meterpreter reverse shell attacks against Windows 11, with one variable changed per round, documented end-to-end with Sysmon telemetry and MITRE ATT&CK mapping.

| Round | Delivery | C2 Port | Key Finding |
|---|---|---|---|
| 1 | Browser (msedge.exe) | 4444 | MotW (ZoneId=3) applied; full Sysmon coverage |
| 2 | PowerShell `Invoke-WebRequest` | 4444 | MotW absent — no SmartScreen warning |
| 3 | PowerShell `Invoke-WebRequest` | 8443 | Port blending into legitimate HTTPS-alt traffic |

**Core conclusion:** Port-based detection alone is insufficient. The reliable detection signal across all three rounds is the *behavioral chain* — unsigned binary spawned by browser or PowerShell, outbound TCP connection, timestomp activity, abrupt process termination.

[→ Phase 1 lab notes](./phase-1-meterpreter-detection/)

### Phase 2 — Wazuh Integration 🚧 In Progress

Deploying Wazuh as a SIEM, replaying Phase 1 attacks against it, and writing custom detection rules for the behavioral signatures identified in Phase 1.

- **Stage A:** Wazuh deployment and SIEM operator skills
- **Stage B:** Detection engineering — custom rules for Phase 1 attack signatures
- **Stage C:** Full SOC simulation — alert triage and incident response documentation

[→ Phase 2 lab notes](./phase-2-wazuh-integration/)

## Tools Used

Nmap · Wireshark · Metasploit / msfvenom · Sysmon (Sysmon64a for ARM64) · Windows Event Viewer · PowerShell · Python HTTP server · Wazuh (Phase 2)

## Notes on the Environment

Windows 11 on UTM runs under Apple Silicon ARM emulation. Sysmon64.exe (x64) cannot fully load its kernel driver in this environment. **Sysmon64a.exe** is the native ARM64 binary and is required for clean driver installation. This is documented in detail in [Day 2](./phase-1-meterpreter-detection/day-2-recon-and-sysmon-troubleshooting.md).

---

*This lab is built for learning and detection research. All offensive activity is contained within isolated virtual machines.*
