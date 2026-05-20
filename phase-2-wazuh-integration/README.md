# Phase 2 — Wazuh Integration

 **In Progress**

## Goal

Deploy Wazuh as a SIEM, integrate it with the existing Sysmon telemetry from Phase 1, replay the three Meterpreter attack rounds against it, and build custom detection rules for the behavioral signatures identified in Phase 1.

## Planned Architecture

| Host | Role | OS |
|---|---|---|
| SOC-WAZUH-01 | Wazuh manager + indexer + dashboard | Ubuntu Server 22.04 LTS |
| SOC-WINDOWS-01 (Non-Prod) | Wazuh agent (existing victim VM) | Windows 11 (ARM64) |
| SOC-ATTACKER-01 | Attacker (existing) | Kali Linux |

## Stages

### Stage A — SIEM Operator Skills
Install Wazuh, deploy the Windows agent, get Sysmon telemetry flowing into the dashboard, learn the navigation.

### Stage B — Detection Engineering
Replay the three Phase 1 attack rounds, identify what Wazuh's default ruleset catches and misses, write custom rules targeting the documented behavioral chain.

### Stage C — Full SOC Simulation
Run blind attack scenarios. Triage from the alert, not from prior knowledge. Produce formal incident reports.

---

*Notes will be added here as each stage completes.*
