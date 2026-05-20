# Day 7 — Wazuh SIEM Deployment (Stage A)

## Objective

Deploy Wazuh as the SIEM for the home lab. Stand up the manager, indexer, and dashboard on a dedicated Ubuntu Server VM. Get the dashboard reachable from the Mac host browser. This is Stage A of Phase 2 — agent deployment and detection engineering happen in later stages.

## Environment

| Host | Role | IP | OS |
|---|---|---|---|
| SOC-WAZUH-01 | Wazuh manager + indexer + dashboard | 192.168.65.6 | Ubuntu Server 24.04.4 LTS (ARM64) |
| SOC-ATTACKER-01 | Attacker (existing) | 192.168.65.3 | Kali Linux |
| SOC-WINDOWS-01 (Non-Prod) | Primary victim (existing, no agent yet) | 192.168.65.4 | Windows 11 (ARM64) |

**Host platform:** Mac Mini M4, 16 GB RAM, 154 GB free disk at start of session.

## VM Provisioning in UTM

Created a new Linux VM in UTM with these allocations:

| Resource | Allocation | Reasoning |
|---|---|---|
| RAM | 4 GB | Wazuh all-in-one minimum. Tight but workable on 16 GB host. |
| CPU | 2 cores | Avoids "Default" (all cores) which contends with macOS. |
| Disk | 64 GB | UTM sparse allocation. Wazuh indexer grows over time, headroom matters. |
| Engine | QEMU + Virtualize | UTM's recommended path. Apple Virtualization is experimental. |
| Network | Shared (192.168.65.0/24) | Same subnet as Kali and Windows victims. |

VM named `SOC-WAZUH-01` to match the existing naming convention.

## Ubuntu Version Decision

Originally planned Ubuntu 22.04 LTS based on Wazuh's documentation. When I checked the official Ubuntu download page, the current default was 26.04 LTS (released April 2026). Wazuh's officially supported list at the time of install was 16.04, 18.04, 20.04, 22.04, 24.04 — 26.04 was not yet officially supported even though community guides existed.

Picked **Ubuntu 24.04 LTS (Noble Numbat)** as the right middle path:

- Officially supported by Wazuh
- Recent enough to be relevant for federal IT work (supported until 2029)
- Mature enough that community troubleshooting exists for edge cases
- Avoids being the early adopter on 26.04 while Wazuh's QA is still landing

**Lesson:** Vendor "supported OS" lists change over time. Always verify against the vendor's current documentation before committing to a version, even when working from recent advice.

## Ubuntu Install Decisions

Walked through the Subiquity installer with deliberate choices:

- **Installer update:** skipped (24.04.4 base installer is fine, kept version consistency)
- **Install type:** Ubuntu Server (full), not minimized — full version includes admin tools needed for hands-on lab work
- **Third-party drivers:** unchecked — virtualized hardware doesn't need proprietary blobs, smaller attack surface
- **Network:** DHCP (assigned 192.168.65.6) — static IP not needed for a learning lab
- **Proxy:** none — direct internet via UTM shared network
- **Mirror:** US (auto-selected, passed tests)
- **Storage:** entire disk, LVM enabled, no LUKS encryption — host Mac has FileVault, LUKS adds boot friction without benefit here
- **LVM expansion:** manually extended root volume from 30 GB default to full 60 GB — Ubuntu installer's default leaves half the volume group unallocated, which would cause indexer storage issues later
- **Ubuntu Pro:** skipped — bookmark for Phase 3 when STIG/FIPS hardening becomes relevant
- **SSH server:** enabled — required for remote administration from Mac
- **Featured snaps:** none — principle of least functionality, only install what's needed

## Post-Install Setup

First boot revealed an installer-leftover issue: the ISO was still attached and the VM attempted to boot back into the installer. Resolved by force-stopping the VM, clearing the CD/DVD entry in UTM settings, and rebooting cleanly to the installed disk.

After clean boot:

1. **Applied 37 pending security updates:**
   ```bash
   sudo apt update && sudo apt upgrade -y
   ```
2. **Established SSH from Mac host:**
   ```bash
   ssh ladi@192.168.65.6
   ```
   Verified host fingerprint on first connect. SSH works because UTM's shared network puts the Mac on 192.168.65.1 and the VMs in the same subnet.

Switching to SSH from the UTM console is a meaningful workflow upgrade — full terminal features, copy/paste, resizable window. Every subsequent command was run over SSH.

## Wazuh Installation

Used Wazuh's official all-in-one installation assistant for version 4.12:

```bash
curl -sO https://packages.wazuh.com/4.12/wazuh-install.sh && sudo bash ./wazuh-install.sh -a
```

The `-a` flag installs manager + indexer + dashboard on a single host. Total install time: roughly 3 minutes on the M4.

Install sequence captured from the log timestamps:

| Time | Stage | Duration |
|---|---|---|
| 23:21:39 | Created `wazuh-install-files.tar` (cluster keys, certs, passwords) | — |
| 23:21:39 → 23:22:30 | Wazuh indexer installed | ~50 sec |
| 23:22:35 → 23:22:37 | Indexer cluster security initialized | seconds |
| 23:22:37 → 23:23:15 | Wazuh manager installed | ~40 sec |
| 23:23:25 → 23:23:34 | Filebeat installed (log shipper) | ~10 sec |
| 23:23:34 → 23:24:10 | Wazuh dashboard installed | ~35 sec |
| 23:24:11 → 23:24:44 | Final config, web app initialization | ~30 sec |

Admin credentials were generated by the installer and saved securely outside the repo.

## Service Verification

Confirmed all three Wazuh services running:

```bash
sudo systemctl status wazuh-indexer wazuh-manager wazuh-dashboard --no-pager
```

All three returned `active (running)`. Notable from the output:

- **wazuh-indexer** (Java process, ~1.4 GB RAM) — OpenSearch fork storing alerts and events
- **wazuh-manager** (~1.0 GB RAM, 154 tasks) — runs the rules engine (`wazuh-analysisd`), file integrity monitor (`wazuh-syscheckd`), agent enrollment (`wazuh-authd`), and active response (`wazuh-execd`)
- **wazuh-dashboard** (~174 MB RAM) — OpenSearch Dashboards web UI

Verified the dashboard was listening on the network:

```bash
sudo ss -tlnp | grep -E ':443|:9200|:55000'
```

Confirmed port 443 (dashboard), 55000 (manager API), and 9200 (indexer, localhost-only — correct).

## Dashboard Access

Accessed the dashboard at `https://192.168.65.6` from the Mac browser. Accepted the self-signed certificate warning (expected for an internally-signed install — Phase 3 candidate to replace with a real certificate).

Logged in with the generated admin credentials. Landed on the Wazuh Overview page showing:

- **Agents Summary:** "No agents registered" — expected, agent deployment is Stage B
- **Last 24 Hours Alerts:** 0 critical, 0 high, 146 medium, 158 low — these are the manager monitoring itself during install, normal baseline noise
- **Endpoint Security modules:** Configuration Assessment, Malware Detection, File Integrity Monitoring, MITRE ATT&CK, Threat Hunting, Vulnerability Detection — all available

## Indicators of a Healthy Install

For future reference if this needs rebuilding:

- All three services show `active (running)` under systemd
- Port 443 listening on `0.0.0.0` (dashboard accessible from any interface)
- Port 55000 listening on `0.0.0.0` (manager API accessible)
- Port 9200 listening on `127.0.0.1` (indexer correctly restricted to localhost)
- Dashboard returns `HTTP/1.1 302 Found` redirecting to `/app/login` on initial GET
- `osd-name: soc-wazuh-01` header confirms identity

## Phase 2 Progress

| Stage | Goal | Status |
|---|---|---|
| A | SIEM operator skills — Wazuh install, dashboard navigation | Complete |
| B | Detection engineering — replay Phase 1 attacks, custom rules | Next |
| C | Full SOC simulation — alert triage, incident reports | After B |

## Notes for Stage B

When Stage B begins, the Wazuh agent will be deployed to **SOC-WINDOWS-01 (Non-Prod)** pointing at manager IP `192.168.65.6` (or hostname once added to local DNS). The Sysmon telemetry from Phase 1 will then flow into Wazuh's indexer instead of just Windows Event Viewer, and custom rules will be written against the behavioral chain documented in Phase 1:

> Browser or PowerShell delivers an executable → unsigned binary with no metadata executes → process initiates outbound TCP connection → process terminates abruptly

The Wazuh manager's default ruleset will catch some of this. The gaps are where custom detection rules get written — that's the real Stage B work.
