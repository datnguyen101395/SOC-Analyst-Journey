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

<img width="1509" height="900" alt="Endpoint active" src="https://github.com/user-attachments/assets/0412f50f-9768-420a-b5d9-f8edc4dc86ab" />
<img width="898" height="481" alt="Agent running" src="https://github.com/user-attachments/assets/531dbc79-74f7-4730-8a34-8adf1772fbd2" />
<img width="784" height="521" alt="Ping Success" src="https://github.com/user-attachments/assets/ad85492c-d4e8-4fa7-9639-1e9646bfa299" />
<img width="1057" height="597" alt="Wazuh manager running" src="https://github.com/user-attachments/assets/a6e9735e-6700-415a-8c9b-364017408570" />
<img width="1284" height="385" alt="Wazuh Indexer running" src="https://github.com/user-attachments/assets/cb04ee3d-39d2-444f-878b-0b30d2132b02" />
<img width="1284" height="357" alt="Wazuh dashboard running" src="https://github.com/user-attachments/assets/75bf9d71-e37c-492b-ad3c-2e9f7ce8fa8a" />

