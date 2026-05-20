# Phase 1 — Meterpreter Detection via Sysmon

Three controlled rounds of Meterpreter reverse shell attacks against a Windows 11 victim, with attacker telemetry captured via Sysmon and analyzed for behavioral detection signatures.

## Experimental Design

Each round changes **one variable** while holding the others constant, isolating the effect of that variable on defender telemetry.

| Round | Delivery Method | C2 Port | Parent Process | MotW Applied? |
|---|---|---|---|---|
| 1 | Browser download (msedge.exe) | 4444 | msedge.exe |  Yes (ZoneId=3) |
| 2 | PowerShell `Invoke-WebRequest` | 4444 | explorer.exe |  No |
| 3 | PowerShell `Invoke-WebRequest` | 8443 | explorer.exe |  No |

## Lab Notes (Chronological)

1. [Day 1 — Network setup](./day-1-network-setup.md)
2. [Day 2 — First network recon and Sysmon troubleshooting](./day-2-recon-and-sysmon-troubleshooting.md)
3. [Day 3 — Recon detection investigation](./day-3-recon-detection-investigation.md)
4. [Day 4 — Round 1: Browser-delivered reverse shell](./day-4-round-1-browser-delivery.md)
5. [Day 5 — Round 2: PowerShell-delivered reverse shell](./day-5-round-2-powershell-delivery.md)
6. [Day 6 — Round 3: C2 port blending (8443)](./day-6-round-3-port-blending.md)

## Key Takeaway

Port-based detection alone is insufficient. The reliable cross-round signal is the **behavioral chain**:

> Browser or PowerShell delivers an executable → unsigned binary with no metadata executes → process initiates outbound TCP connection → process terminates abruptly

This behavior pattern persists regardless of delivery method or destination port, and forms the basis for the custom Wazuh rules built in Phase 2.
