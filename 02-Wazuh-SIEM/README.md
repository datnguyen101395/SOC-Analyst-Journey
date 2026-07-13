# Lab 02 - Wazuh SIEM Deployment

## Objective

Deploy an enterprise-style Wazuh SIEM environment and enroll a Windows 11 endpoint for centralized security monitoring.

---

## Environment

| Component | Version |
|----------|---------|
| Ubuntu Server | 24.04 LTS |
| Wazuh | 4.13.1 |
| Windows | Windows 11 Pro |
| Virtualization | Parallels Desktop |
| Host | MacBook Pro M4 Pro |

---

## Architecture

Windows 11 Endpoint

↓

Wazuh Agent

↓

Wazuh Manager

↓

Wazuh Indexer

↓

Wazuh Dashboard

---

## Tasks Completed

- Installed Ubuntu Server
- Installed Wazuh Manager
- Installed Wazuh Indexer
- Installed Wazuh Dashboard
- Verified Linux services
- Verified network connectivity
- Installed Wazuh Agent
- Registered Windows endpoint
- Verified endpoint communication

---

## Verification

### Verify Wazuh Services

```bash
systemctl status wazuh-manager
systemctl status wazuh-indexer
systemctl status wazuh-dashboard
```

Expected:

```
Active (running)
```

---

### Verify Connectivity

Windows:

```cmd
ping 192.168.64.2
```

Expected:

```
Reply from 192.168.64.2
```

---

### Verify Windows Agent

```powershell
Get-Service Wazuh
```

Expected:

```
Running
```

---

### Verify Dashboard

Endpoints →

WIN11-LAB →

Status: Active

---

## Skills Learned

- Linux service management
- Wazuh architecture
- Endpoint enrollment
- Network troubleshooting
- Windows PowerShell
- Basic SOC infrastructure

---

## Lessons Learned

- Verify connectivity before installing software.
- Understand the data flow instead of memorizing commands.
- Every component has one responsibility:
  - Agent → Collect
  - Manager → Analyze
  - Indexer → Store
  - Dashboard → Visualize

---

## Screenshots

(Add screenshots here)
