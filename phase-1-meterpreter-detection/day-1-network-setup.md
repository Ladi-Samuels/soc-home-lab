# Day 1 — Network Setup

## Objective

Build the foundational lab environment: two virtual machines that can communicate over a shared network.

## Environment

| Host | OS | IP | RAM | CPU | Storage |
|---|---|---|---|---|---|
| SOC-ATTACKER-01 | Kali Linux | 192.168.65.3 | 4 GB | 2 cores | 40 GB |
| SOC-WINDOWS-02 | Windows 11 | 192.168.65.4 | 4 GB | 2 cores | 15.1 GB |

**Network mode:** Shared network (UTM)

## Activities

- Created both VMs in UTM on Mac Mini M4
- Configured shared networking
- Verified bidirectional connectivity via ping

## Result

Ping successful from both devices. Baseline lab connectivity confirmed.

> **Note:** Day 1 working notes originally recorded Windows 11 as `192.168.65.1`. The correct address used throughout the rest of the lab is `192.168.65.4`, confirmed via Days 2–6 telemetry.
