# Lab 04 — Sysmon Integration and Raw Telemetry Hunting

## Objective

Deploy Microsoft Sysmon on a Windows 11 endpoint, integrate the Sysmon event channel with Wazuh, and make raw endpoint telemetry searchable even when an event does not trigger a detection rule.

---

## Lab Overview

Windows Event Logs provide valuable security information, but they do not always include the detailed process context required for threat hunting and incident investigation.

In this lab, I installed Microsoft Sysmon to improve endpoint visibility. Sysmon records detailed telemetry such as process creation, command-line arguments, parent processes, executable hashes, network connections, DNS activity, file creation, and registry changes.

I then configured the Wazuh agent to collect the Sysmon Operational event channel. During testing, I discovered that successfully collected telemetry does not automatically appear as a Wazuh alert. Events only appear in the normal alert index when they match a detection rule.

To search all collected telemetry, including benign activity such as `notepad.exe`, I enabled Wazuh JSON archives, configured Filebeat to forward archived events, and created a `wazuh-archives-*` data view in the Wazuh Dashboard.

---

## Architecture

```text
Windows Process Execution
        ↓
Microsoft Sysmon
        ↓
Sysmon Operational Event Log
        ↓
Wazuh Agent
        ↓
Wazuh Manager
        ↓
archives.json
        ↓
Filebeat
        ↓
Wazuh Indexer
        ↓
wazuh-archives-*
        ↓
Wazuh Discover
        ↓
SOC Analyst Investigation
```

---

## Environment

| Component | Technology |
|---|---|
| Host System | MacBook Pro with Apple Silicon |
| Virtualization | Parallels Desktop / UTM |
| Endpoint | Windows 11 ARM64 |
| SIEM Server | Ubuntu Server |
| SIEM Platform | Wazuh |
| Endpoint Telemetry | Microsoft Sysmon |
| Sysmon Configuration | SwiftOnSecurity |
| Log Forwarding | Wazuh Agent and Filebeat |

---

## Tasks Completed

- Downloaded and extracted Microsoft Sysmon.
- Identified `Sysmon64a.exe` as the correct ARM64 executable.
- Downloaded the SwiftOnSecurity Sysmon configuration.
- Verified that the configuration file contained valid XML.
- Installed Sysmon as a Windows service and driver.
- Confirmed Sysmon events were recorded locally.
- Added the Sysmon Operational channel to the Wazuh agent configuration.
- Restarted and verified the Wazuh agent service.
- Generated process creation telemetry.
- Confirmed that `net.exe` generated a Wazuh alert.
- Confirmed that `notepad.exe` was logged locally but did not generate an alert.
- Enabled Wazuh JSON archive logging.
- Enabled Filebeat archive forwarding.
- Created the `wazuh-archives-*` index pattern.
- Generated a unique PowerShell test event.
- Located and investigated the raw Sysmon event in Wazuh Discover.

---

## Sysmon Installation

Sysmon was extracted to:

```text
C:\Tools\Sysmon
```

Because the Windows VM uses ARM64 architecture, the correct executable was:

```text
Sysmon64a.exe
```

The Sysmon configuration was downloaded and verified before installation.

Installation command:

```powershell
.\Sysmon64a.exe -accepteula -i .\sysmonconfig-export.xml
```

Service verification:

```powershell
Get-Service *Sysmon*
```

Expected result:

```text
Status: Running
```

---

## Sysmon Event Channel

Sysmon writes telemetry to:

```text
Applications and Services Logs
→ Microsoft
→ Windows
→ Sysmon
→ Operational
```

Important Sysmon Event IDs include:

| Event ID | Description |
|---|---|
| 1 | Process creation |
| 3 | Network connection |
| 11 | File creation |
| 13 | Registry value modification |
| 22 | DNS query |

---

## Wazuh Agent Configuration

The following block was added to the Windows Wazuh agent configuration:

```xml
<localfile>
  <location>Microsoft-Windows-Sysmon/Operational</location>
  <log_format>eventchannel</log_format>
</localfile>
```

The Wazuh agent configuration file is located at:

```text
C:\Program Files (x86)\ossec-agent\ossec.conf
```

The agent service was restarted and verified:

```powershell
Restart-Service WazuhSvc
Get-Service WazuhSvc
```

---

## Alerts Versus Raw Telemetry

One of the most important lessons from this lab was understanding the difference between collected telemetry and generated alerts.

```text
Collected telemetry ≠ Alert generated
```

A Sysmon event may be:

1. Recorded locally by Sysmon.
2. Collected by the Wazuh agent.
3. Received by the Wazuh manager.
4. Stored as raw telemetry.

However, it only appears in the normal Wazuh alert index if a detection rule matches it.

Example:

```text
net.exe
→ Sysmon Event ID 1
→ Wazuh rule matched
→ Alert visible in Threat Hunting
```

```text
notepad.exe
→ Sysmon Event ID 1
→ No detection rule matched
→ No alert in Threat Hunting
→ Still searchable through raw archives
```

---

## Enabling Raw Wazuh Archives

Before modifying the manager configuration, a backup was created:

```bash
sudo cp /var/ossec/etc/ossec.conf /var/ossec/etc/ossec.conf.backup-before-archives
```

Inside the existing `<global>` section, raw JSON logging was enabled:

```xml
<logall_json>yes</logall_json>
```

The Wazuh manager was restarted:

```bash
sudo systemctl restart wazuh-manager
systemctl is-active wazuh-manager
```

The archive file was verified:

```bash
sudo ls -lh /var/ossec/logs/archives/archives.json
```

---

## Filebeat Archive Forwarding

The Filebeat configuration was backed up:

```bash
sudo cp /etc/filebeat/filebeat.yml /etc/filebeat/filebeat.yml.backup-before-archives
```

Archive forwarding was enabled:

```yaml
archives:
  enabled: true
```

Filebeat was restarted and verified:

```bash
sudo systemctl restart filebeat
systemctl is-active filebeat
```

---

## Wazuh Archive Index

The archive index was verified with:

```bash
curl -k -u admin "https://127.0.0.1:9200/_cat/indices/wazuh-archives-*?v"
```

A new dashboard data view was created:

```text
wazuh-archives-*
```

The time field selected was:

```text
timestamp
```

This made all archived events searchable through Wazuh Discover.

---

## Controlled Sysmon Test

A unique PowerShell process event was generated:

```powershell
powershell.exe -NoProfile -Command "Write-Output 'SOC-LAB-SYSMON-TEST-001'"
```

The event was first verified locally:

```powershell
Get-WinEvent `
  -LogName "Microsoft-Windows-Sysmon/Operational" `
  -FilterXPath "*[System[(EventID=1)]]" `
  -MaxEvents 20 |
  Where-Object { $_.Message -match "SOC-LAB-SYSMON-TEST-001" } |
  Format-List TimeCreated, Id, Message
```

The same event was then located in:

```text
Wazuh Dashboard
→ Explore
→ Discover
→ wazuh-archives-*
```

Search term:

```text
SOC-LAB-SYSMON-TEST-001
```

---

## Fields Investigated

The expanded Sysmon event contained detailed process information including:

```text
data.win.system.eventID
data.win.system.channel
data.win.eventdata.image
data.win.eventdata.commandLine
data.win.eventdata.parentImage
data.win.eventdata.user
data.win.eventdata.hashes
data.win.eventdata.processGuid
```

These fields answer important investigative questions:

| Question | Relevant Field |
|---|---|
| What executed? | `image` |
| What command was run? | `commandLine` |
| What launched it? | `parentImage` |
| Who executed it? | `user` |
| What file was executed? | `hashes` |
| Which process instance was involved? | `processGuid` |

---

## Troubleshooting

### Wazuh Agent Failed After Configuration Change

After editing the Windows Wazuh agent configuration, the service stopped.

The original configuration backup was restored, which confirmed that the new XML block caused the failure.

The Sysmon `<localfile>` block was then added carefully inside the correct `<ossec_config>` section.

Lesson:

> Configuration backups should be created before modifying production or security-service configuration files.

---

### Sysmon Events Were Missing from Threat Hunting

`notepad.exe` appeared in the local Sysmon Operational log but not in Wazuh Threat Hunting.

At first, this appeared to be an ingestion problem. Further investigation showed that Threat Hunting displayed events from the alert index, not all collected telemetry.

Because `notepad.exe` did not match a Wazuh rule, no alert was generated.

Resolution:

- Enabled `logall_json`.
- Enabled Filebeat archive forwarding.
- Created the `wazuh-archives-*` data view.
- Searched the event through Discover.

---

### Incorrect Search Filters

A previously applied filter made valid telemetry appear to be missing.

Resolution:

- Removed all existing filters.
- Expanded the time range.
- Started with a broad agent filter.
- Narrowed the search by event channel, Event ID, and process name.

Lesson:

> A restrictive SIEM filter can make healthy telemetry appear unavailable.

---

## Skills Demonstrated

- Microsoft Sysmon deployment
- Windows ARM64 tool selection
- XML configuration review
- Windows service management
- Wazuh agent configuration
- Linux configuration management
- Filebeat configuration
- Raw log archiving
- Wazuh index management
- Endpoint telemetry analysis
- Process-tree investigation
- SIEM troubleshooting
- Threat hunting

---

## Key Lessons Learned

- Sysmon provides much richer endpoint visibility than default Windows logs.
- Sysmon writes events to Windows Event Logs; it does not send them directly to Wazuh.
- The Wazuh agent collects the Sysmon Operational channel.
- A collected event does not automatically become an alert.
- Detection rules determine which events are promoted into the alert index.
- Raw archive data is valuable for proactive threat hunting.
- Command lines and parent-child process relationships are critical during investigations.
- Backups should always be created before modifying configuration files.
- Search filters should be cleared before assuming telemetry is missing.
- Threat hunting often begins with telemetry that did not trigger an alert.

---

## Screenshots
<img width="1118" height="628" alt="01-sysmon-extracted-files" src="https://github.com/user-attachments/assets/a1ea18f2-7e7c-4276-a856-7f4770828c2f" />
<img width="1335" height="584" alt="02-sysmon-config-verified" src="https://github.com/user-attachments/assets/89f9272e-8e02-4fd1-957a-32143ed0368c" />
<img width="970" height="246" alt="03-sysmon-service-running" src="https://github.com/user-attachments/assets/5a4f249f-8cb8-4652-b434-ec1c102e7c74" />
<img width="984" height="713" alt="04-Local-Event-Analysis" src="https://github.com/user-attachments/assets/0019c17c-0b27-481b-83c0-2b4e9e88e0a4" />
<img width="910" height="780" alt="05-wazuh-agent-sysmon-config" src="https://github.com/user-attachments/assets/0266644e-6190-47e9-bba8-2fe29d73872b" />
<img width="969" height="609" alt="06-wazuh-agent-service-running" src="https://github.com/user-attachments/assets/c3f9c897-f3e8-4531-80de-58701642ce9e" />
<img width="1493" height="804" alt="07-sysmon-event-in-wazuh" src="https://github.com/user-attachments/assets/766ff1c1-463a-4601-8ef4-de9d8f9d69b2" />
<img width="1254" height="756" alt="08-Expanded-sysmon-event-details" src="https://github.com/user-attachments/assets/f0dd0fb5-60d9-47d9-8b4a-f18540f4ddd8" />



## Interview Talking Points

If asked about this project, I would explain:

- Why Sysmon provides stronger endpoint visibility than default Windows logging.
- How Sysmon integrates with Windows Event Logs and the Wazuh agent.
- The difference between Wazuh alert data and raw archived telemetry.
- How I identified why `notepad.exe` did not appear in Threat Hunting.
- How I enabled `wazuh-archives-*` to search all collected telemetry.
- How I investigated process image, command line, parent process, user, hashes, and Process GUID.
- How configuration backups helped me recover from a failed Wazuh agent service.
- How following the telemetry pipeline helped isolate the issue.

---

## What I Would Do in a Production SOC

If this activity appeared in a production environment, I would:

- Determine whether the executed process was expected.
- Review the full command line for suspicious arguments.
- Analyze the parent-child process relationship.
- Verify the user account responsible for the activity.
- Compare executable hashes against trusted sources and threat intelligence.
- Search for related process, network, DNS, file, and registry events.
- Identify similar behavior across other endpoints.
- Escalate or contain the endpoint if the behavior appeared malicious.

---

## Next Steps

- Create a custom Wazuh detection rule.
- Generate suspicious PowerShell behavior.
- Trigger an automated Wazuh alert.
- Map the activity to MITRE ATT&CK.
- Begin detection engineering and threat-hunting exercises.
