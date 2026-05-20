# Day 2 — First Network Recon and Sysmon Troubleshooting

## Objective

Install Sysmon on the Windows 11 victim to enable endpoint telemetry, and perform initial network reconnaissance from Kali.

## The Sysmon Installation Problem

While attempting to install Sysmon on the Windows 11 target, the service was blocked and access was denied. Began troubleshooting against two common root causes:

1. A blocked or unsigned driver
2. An incomplete prior installation stuck in the registry

### Troubleshooting Attempts

| Step | Action | Result |
|---|---|---|
| 1 | Ran `sysmon64.exe -u` to remove the broken installation | Did not resolve |
| 2 | Disabled the Microsoft Vulnerable Driver Blocklist | Did not resolve |
| 3 | Ran `Enable-WindowsOptionalFeature -Online -FeatureName Sysmon` | Returned "feature unknown" error |
| 4 | Attempted forced uninstall and reinstall | Did not resolve |
| 5 | Created missing registry key `HypervisorEnforcedCodeIntegrity` set to `0` | Access still denied |
| 6 | Disabled Secure Boot | Did not resolve |

### Root Cause

Windows 11 on UTM runs under Apple Silicon (ARM architecture).
- `Sysmon64.exe` is x64 — its kernel driver cannot fully load under ARM emulation
- `Sysmon64a.exe` is the **native ARM64 binary** — driver loads and starts cleanly

### Resolution

The issue was resolved by installing using the appropriate binary for the host architecture. In some emulated environments, the 32-bit `sysmon.exe` also has better compatibility than `sysmon64.exe`.

## Network Recon Notes

After Sysmon was resolved, attempted reconnaissance from Kali:

- `ping` to Windows 11 host: **failed** (ICMP filtered)
- Default `nmap` scan: assumed host was down
- `nmap -Pn -F` against the target: bypassed ICMP discovery and received responses
- **Result:** Host alive · 99 TCP ports filtered · Port **5357/tcp open (wsdapi)**

## Lessons Learned

- Hosts may block ICMP but still be online and reachable
- Firewall configuration heavily affects reconnaissance results
- Nmap host discovery and port scanning are distinct phases — `-Pn` is essential when ICMP is filtered
- Driver architecture matching matters in emulated environments; always verify the binary matches the host CPU architecture, not the guest OS bitness
