# Lab 05 — Networking Fundamentals for SOC Analysts

## Objective

The goal of this lab was to document the network layout of my home lab, verify connectivity between my Windows endpoint and Ubuntu Wazuh server, identify listening services, and compare local listening ports with ports reachable from another host.

This lab helped me practice basic networking concepts that are important for SOC analysis, including:

- IP addresses
- Subnet masks
- Default gateways
- Host-to-host connectivity
- Listening ports
- Open services
- Port scanning
- Basic network troubleshooting

---

## Lab Environment

| Component | System |
|---|---|
| Host computer | MacBook Pro |
| Windows endpoint | Windows 11 ARM64 |
| Windows IP address | 10.211.55.5 |
| Ubuntu server | Ubuntu Server |
| Ubuntu IP address | 192.168.64.2 |
| SIEM platform | Wazuh |
| Scanning tool | Nmap |
| Virtualization | Parallels / UTM on macOS |

---

## Lab Architecture

```
Windows endpoint
10.211.55.5
        ↓
Network connectivity test
        ↓
Ubuntu Wazuh server
192.168.64.2
        ↓
Listening services
        ↓
Port analysis
```

The Windows endpoint and Ubuntu server were on different virtual networks, but connectivity was still possible through the virtual networking configuration.

---

## Tasks Completed

During this lab, I completed the following tasks:

- Identified the Windows endpoint IP address using `ipconfig`.
- Identified the Ubuntu server IP address using `ip -br addr`.
- Verified connectivity from Windows to Ubuntu using `ping`.
- Reviewed listening ports on the Ubuntu server using `ss`.
- Installed and used Nmap on the Windows endpoint.
- Ran a basic Nmap scan against the Ubuntu server.
- Ran a service/version Nmap scan against the Ubuntu server.
- Compared local listening ports with network-visible ports.
- Documented expected services and their purpose.

---

## Windows Endpoint Network Information

On the Windows endpoint, I used the following command:

```powershell
ipconfig
```

The Windows endpoint was assigned the following network information:

| Field | Value |
|---|---|
| IPv4 Address | 10.211.55.5 |
| Subnet Mask | 255.255.255.0 |
| Default Gateway | 10.211.55.1 |
| DNS Suffix | localdomain |

<img width="882" height="451" alt="01-windows-ipconfig" src="https://github.com/user-attachments/assets/b2050bfc-91d3-4271-a6f4-94c029392d4a" />


This confirmed that the Windows endpoint was connected to the Parallels virtual network.

---

## Ubuntu Server Network Information

On the Ubuntu Wazuh server, I used the following command:

```bash
ip -br addr
```

The Ubuntu server was assigned the following network information:

| Field | Value |
|---|---|
| Interface | enp0s1 |
| Status | UP |
| IPv4 Address | 192.168.64.2/24 |
| Loopback Address | 127.0.0.1/8 |

<img width="1095" height="85" alt="02-ubuntu-ip-address" src="https://github.com/user-attachments/assets/b29d6783-d494-4366-987e-6e2bbdce9c04" />


This confirmed that the Ubuntu server had an active network interface and was reachable at:

```text
192.168.64.2
```

---

## Connectivity Test

From the Windows endpoint, I tested connectivity to the Ubuntu Wazuh server using `ping`.

Command used:

```powershell
ping 192.168.64.2
```

Result:

| Test | Result |
|---|---|
| Packets Sent | 4 |
| Packets Received | 4 |
| Packet Loss | 0% |
| Minimum Response Time | 0 ms |
| Maximum Response Time | 2 ms |
| Destination | 192.168.64.2 |

<img width="645" height="277" alt="03-windows-ping-ubuntu" src="https://github.com/user-attachments/assets/eef438f3-ec73-47ee-9c5f-a0c1617a88db" />


This confirmed that the Windows endpoint could successfully communicate with the Ubuntu Wazuh server.

---

## Listening Ports on Ubuntu

On the Ubuntu Wazuh server, I used the `ss` command to view listening TCP and UDP ports.

Command used:

```bash
sudo ss -tulnp
```

The command options mean:

| Option | Meaning |
|---|---|
| `-t` | Show TCP sockets |
| `-u` | Show UDP sockets |
| `-l` | Show listening ports |
| `-n` | Show numeric addresses and ports |
| `-p` | Show the process using the port |

<img width="1277" height="372" alt="04-ubuntu-listening-ports" src="https://github.com/user-attachments/assets/ab78b072-bc30-46de-951b-db0d2b2e7cc0" />


Important listening ports observed:

| Port | Protocol | Process / Service | Purpose |
|---|---|---|---|
| 22 | TCP | SSH / systemd | Remote administration |
| 443 | TCP | node | Wazuh Dashboard over HTTPS |
| 1514 | TCP | wazuh-remoted | Wazuh agent event forwarding |
| 1515 | TCP | wazuh-authd | Wazuh agent enrollment |
| 55000 | TCP | python3 | Wazuh API |
| 9200 | TCP | java | Wazuh Indexer API |
| 9300 | TCP | java | Wazuh Indexer internal communication |
| 53 | TCP/UDP | systemd-resolve | Local DNS resolver |
| 68 | UDP | systemd-networkd | DHCP client |
| 323 | UDP | chronyd | Time synchronization |

This showed which services were listening locally on the Ubuntu server.

---

## Basic Nmap Scan

From the Windows endpoint, I ran a basic Nmap scan against the Ubuntu Wazuh server.

Command used:

```powershell
nmap 192.168.64.2
```

<img width="682" height="223" alt="05-window-ports-connectivity-test" src="https://github.com/user-attachments/assets/edc9c57f-86d5-4bf2-9795-5e7db806760d" />


The scan identified the following open ports from the Windows endpoint:

| Port | State | Service |
|---|---|---|
| 22/tcp | open | ssh |
| 443/tcp | open | https |

Nmap also showed that 998 TCP ports were closed.

This confirmed that the Ubuntu server was reachable from the Windows endpoint and that SSH and HTTPS were exposed over the network.

---

## Nmap Service Scan

I then ran a service/version scan using Nmap.

Command used:

```powershell
nmap -sV 192.168.64.2
```

The `-sV` option attempts to identify the service and version running behind open ports.

<img width="1557" height="867" alt="06-nmap-service-scan" src="https://github.com/user-attachments/assets/2be93895-2423-4f68-9682-ced5567c9a8d" />


Results:

| Port | State | Service | Finding |
|---|---|---|---|
| 22/tcp | open | ssh | OpenSSH 10.2p1 Ubuntu |
| 443/tcp | open | ssl/https | HTTPS service detected |

Nmap identified the SSH service version. It also confirmed that port 443 was running HTTPS, which was expected because the Wazuh Dashboard uses HTTPS.

Nmap produced a long service fingerprint for port 443. This happened because the service responded to Nmap probes, but Nmap did not fully identify the exact application behind the HTTPS service.

The important finding was:

```text
443/tcp open ssl/https
```

---

## Port Analysis

I compared the local listening ports from `ss` with the ports visible from the Windows endpoint through Nmap.

<img width="906" height="511" alt="07-port-analysis-table" src="https://github.com/user-attachments/assets/2d571d29-8142-4eb8-adb9-c752af7311c1" />


Summary:

| Port | Seen in `ss`? | Seen in Nmap? | Expected? | Purpose |
|---|---|---|---|---|
| 22/tcp | Yes | Yes | Yes | SSH remote administration |
| 443/tcp | Yes | Yes | Yes | Wazuh Dashboard over HTTPS |
| 1514/tcp | Yes | No | Yes | Wazuh agent event forwarding |
| 1515/tcp | Yes | No | Yes | Wazuh agent enrollment |
| 55000/tcp | Yes | No | Yes | Wazuh API |
| 9200/tcp | Yes | No | Yes | Wazuh Indexer API |
| 9300/tcp | Yes | No | Yes | Wazuh Indexer internal communication |

Important finding:

```text
The Ubuntu server had several services listening locally, but only ports 22 and 443 were reachable from the Windows endpoint during the Nmap scan.
```

This showed that local listening ports and externally reachable ports are not always the same.

---

## Difference Between `ss` and Nmap

One of the main lessons from this lab was understanding the difference between local service visibility and network visibility.

```text
ss -tulnp
= What the server is listening on locally
```

```text
nmap 192.168.64.2
= What another host can reach over the network
```

For example, the Ubuntu server showed several Wazuh-related services listening locally. However, from the Windows endpoint, only ports 22 and 443 were visible through Nmap.

This is important because a SOC analyst may need to determine whether a service is only local to the system or exposed to other hosts on the network.

---

## Expected Services

The open ports found during this lab were expected for the Ubuntu Wazuh server.

### Port 22 — SSH

SSH is used for remote administration of the Ubuntu server.

Analyst questions:

- Is SSH expected on this server?
- Who should be allowed to connect?
- Is password login enabled?
- Is the service exposed outside the trusted network?
- Are there failed login attempts?

### Port 443 — HTTPS

HTTPS is used to access the Wazuh Dashboard.

Analyst questions:

- Is this web service expected?
- Is it using HTTPS?
- Does it require authentication?
- Is the certificate trusted?
- Should the dashboard be reachable from this network?

### Port 1514 — Wazuh Agent Forwarding

Wazuh agents use this port to send event data to the Wazuh manager.

Analyst questions:

- Are agents successfully forwarding logs?
- Are unknown agents connecting?
- Is the port exposed only where needed?

### Port 1515 — Wazuh Agent Enrollment

This port is used when registering agents with the Wazuh manager.

Analyst questions:

- Is enrollment expected to be open?
- Should it be restricted after agent registration?
- Are unauthorized systems attempting to enroll?

### Port 55000 — Wazuh API

This is used for Wazuh API communication.

Analyst questions:

- Is API access restricted?
- Are authentication failures occurring?
- Is the API exposed beyond the lab network?

### Ports 9200 and 9300 — Wazuh Indexer

These ports are related to the Wazuh indexer.

Analyst questions:

- Are these ports exposed externally?
- Should they be limited to localhost or trusted systems?
- Are there unauthorized connection attempts?

---

## Troubleshooting Notes

### Nmap Was Not Installed

At first, the `nmap` command was not recognized in Windows PowerShell.

This meant Nmap was not installed or not available in the system PATH.

Resolution:

- Installed Nmap on the Windows endpoint.
- Opened a new PowerShell window.
- Confirmed that the `nmap` command worked.
- Ran scans against the Ubuntu server.

Lesson learned:

```text
If a command is not recognized, check whether the tool is installed and whether it is available in PATH.
```

---

### Different Results Between `ss` and Nmap

The `ss` command showed many listening services on Ubuntu, but Nmap from Windows only showed ports 22 and 443.

At first, this could look confusing. The important lesson was that these tools answer different questions.

```text
ss answers:
What is listening on this machine?
```

```text
Nmap answers:
What can another machine reach?
```

This difference can happen because services may be bound to localhost, restricted by firewall rules, filtered by network routing, or not reachable from the scanning host.

---

### Service Detection Fingerprint on Port 443

The Nmap service scan produced a long fingerprint for port 443.

This happened because Nmap detected an HTTPS service but could not fully match it to a specific known application fingerprint.

This was not a failure. The important result was that port 443 was open and responding as HTTPS.

---

## SOC Analyst Mindset

This lab helped me practice the basic investigation questions a SOC analyst should ask when reviewing network activity:

```text
What is the source IP?
What is the destination IP?
What port is being used?
What service normally uses that port?
Is the service expected on this system?
Is the port reachable from another host?
Is the service exposed more than necessary?
Does the process using the port match the expected service?
```

In this lab:

```text
Source host: Windows endpoint
Source IP: 10.211.55.5

Destination host: Ubuntu Wazuh server
Destination IP: 192.168.64.2

Reachable ports:
22/tcp
443/tcp
```

Both ports were expected for this lab environment.

---

## Skills Demonstrated

This lab helped me practice:

- Windows network information review
- Linux network interface review
- Basic host connectivity testing
- Ping troubleshooting
- Linux listening port analysis
- Nmap scanning
- Service/version detection
- Port analysis
- Comparing local and network-visible services
- Basic SOC network investigation
- Documentation of expected services

---

## Key Lessons Learned

- `ipconfig` shows Windows network information.
- `ip addr` shows Linux network interface information.
- `ping` tests basic reachability between hosts.
- `ss -tulnp` shows services listening locally on a Linux system.
- Nmap shows which ports are reachable from another host.
- A service can listen locally but not be reachable from another machine.
- Port 22 is commonly used for SSH.
- Port 443 is commonly used for HTTPS.
- Open ports should be reviewed to determine whether they are expected.
- Service/version scans provide more detail than basic port scans.
- Different tools answer different network investigation questions.
- In a SOC investigation, identifying the source IP, destination IP, port, service, and expected behavior is a basic but important skill.

---

## Next Steps

The next lab will build on this by using Kali Linux to perform basic scanning and begin attacker-versus-defender testing in a controlled lab environment.
