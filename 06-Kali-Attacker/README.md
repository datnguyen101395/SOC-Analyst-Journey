# Lab 06 — Kali Linux Basics and Safe Scanning

## Objective

The goal of this lab was to introduce Kali Linux as a controlled attacker/testing machine in my home lab. I verified Kali's network configuration, tested connectivity to the Ubuntu Wazuh server, performed basic Nmap scans, and compared the scan results with the previous Windows endpoint scans.

This lab focused on safe, authorized scanning only against my own lab system.

---

## Lab Environment

| Component | System |
|---|---|
| Host computer | MacBook Pro |
| Attacker/testing machine | Kali Linux |
| Kali IP address | 192.168.64.4 |
| Target system | Ubuntu Wazuh server |
| Target IP address | 192.168.64.2 |
| SIEM platform | Wazuh |
| Scanning tool | Nmap |
| Virtualization | UTM / macOS |

---

## Lab Architecture

```text
Kali Linux VM
192.168.64.4
        ↓
Ping / Nmap scans
        ↓
Ubuntu Wazuh Server
192.168.64.2
        ↓
Open services
22/tcp SSH
443/tcp HTTPS
```

Kali and the Ubuntu Wazuh server were both on the same virtual network:

```text
192.168.64.0/24
```

This allowed Kali to directly communicate with the Wazuh server.

---

## Tasks Completed

During this lab, I completed the following tasks:

- Logged into Kali Linux.
- Identified Kali's IP address using `ip -br addr`.
- Verified that Kali could reach the Ubuntu Wazuh server using `ping`.
- Ran a basic Nmap scan from Kali against the Wazuh server.
- Ran an Nmap service/version scan from Kali.
- Identified exposed services on the Wazuh server.
- Compared Kali scan results with previous Windows Nmap scan results.
- Documented the findings from an analyst perspective.

---

## Kali Network Information

On Kali Linux, I used the following command to identify the network interface and IP address:

```bash
ip -br addr
```

The output showed:

| Field | Value |
|---|---|
| Interface | eth0 |
| Status | UP |
| IPv4 Address | 192.168.64.4/24 |
| Network | 192.168.64.0/24 |

<img width="702" height="306" alt="01-kali-ip-address" src="https://github.com/user-attachments/assets/a8dfc94d-740e-478e-9485-dccb25017475" />


This confirmed that the Kali VM had an active network interface and was assigned the IP address:

```text
192.168.64.4
```

---

## Connectivity Test

From Kali, I tested connectivity to the Ubuntu Wazuh server.

Command used:

```bash
ping -c 4 192.168.64.2
```

The result showed:

| Test | Result |
|---|---|
| Packets Transmitted | 4 |
| Packets Received | 4 |
| Packet Loss | 0% |
| Destination | 192.168.64.2 |

<img width="690" height="239" alt="02-Kali-ping-wazuh-server" src="https://github.com/user-attachments/assets/9c96dbba-74fb-43d9-b0e2-c4139dece0c2" />


This confirmed that the Kali VM could successfully communicate with the Ubuntu Wazuh server.

---

## Basic Nmap Scan

From Kali, I ran a basic Nmap scan against the Ubuntu Wazuh server.

Command used:

```bash
nmap 192.168.64.2
```

<img width="690" height="193" alt="03-kali-nmap-basic-scan" src="https://github.com/user-attachments/assets/850e9864-4381-4b1b-9d2b-70aa18f43ecc" />


The scan found the following open ports:

| Port | State | Service |
|---|---|---|
| 22/tcp | open | ssh |
| 443/tcp | open | https |

Nmap also showed that 998 TCP ports were closed.

This confirmed that the Wazuh server was reachable from Kali and that SSH and HTTPS were exposed over the network.

---

## Nmap Service Scan

I then ran a service/version scan using Nmap.

Command used:

```bash
nmap -sV 192.168.64.2
```

The `-sV` option attempts to identify the service and version running behind open ports.

<img width="573" height="587" alt="04-kali-nmap-service-scan" src="https://github.com/user-attachments/assets/591bc7d3-8958-41b7-960a-7906f4ca2d39" />


Results:

| Port | State | Service | Version / Finding |
|---|---|---|---|
| 22/tcp | open | ssh | OpenSSH 10.2p1 Ubuntu |
| 443/tcp | open | ssl/https | HTTPS service detected |

Nmap was able to identify the SSH service version on port 22.

For port 443, Nmap confirmed that an HTTPS service was running. The scan produced a long service fingerprint because Nmap received responses from the web service but did not fully identify the exact application.

In this lab, port 443 was expected because the Wazuh Dashboard is accessed over HTTPS.

---

## Scan Comparison

I compared the Nmap results from the Kali VM with the previous scan from the Windows endpoint.

<img width="926" height="709" alt="05-kali-scan-comparison" src="https://github.com/user-attachments/assets/2a1497b1-7f17-4f61-b54b-5d104f959b6c" />


Target:

| Host | IP Address |
|---|---|
| Ubuntu Wazuh Server | 192.168.64.2 |

Source hosts:

| Source Host | IP Address | Role |
|---|---|---|
| Windows Endpoint | 10.211.55.5 | Monitored endpoint |
| Kali VM | 192.168.64.4 | Attacker/testing machine |

Nmap results:

| Source Host | Command | Open Ports Found |
|---|---|---|
| Windows Endpoint | `nmap 192.168.64.2` | 22/tcp, 443/tcp |
| Kali VM | `nmap 192.168.64.2` | 22/tcp, 443/tcp |

Both systems identified the same exposed services on the Ubuntu Wazuh server.

This showed that the server's visible attack surface was consistent from both the monitored Windows endpoint and the Kali testing machine.

---

## Expected Services

The open ports found in this lab were expected for the Ubuntu Wazuh server.

### Port 22 — SSH

SSH is used for remote administration of the Ubuntu server.

From an analyst perspective, I would ask:

- Is SSH expected on this server?
- Who should be allowed to connect?
- Is password login enabled?
- Is SSH exposed outside the trusted network?
- Are there failed login attempts?

### Port 443 — HTTPS

HTTPS is used to access the Wazuh Dashboard.

From an analyst perspective, I would ask:

- Is this web service expected?
- Is it using HTTPS?
- Does it require authentication?
- Should the dashboard be reachable from this network?
- Are there suspicious login attempts?

---

## Difference Between Connectivity and Scanning

One important lesson from this lab was the difference between testing connectivity and scanning services.

```text
ping
= Can the host respond over the network?
```

```text
nmap
= What ports and services are reachable on the host?
```

The ping test confirmed that Kali could reach the Ubuntu Wazuh server.

The Nmap scan identified which TCP ports were open and what services were likely running behind them.

---

## Analyst Interpretation

The basic investigation questions for this lab were:

```text
What is the source host?
What is the source IP?
What is the destination host?
What is the destination IP?
Which ports are open?
Which services are running?
Are those services expected?
Do different source hosts see the same exposed ports?
```

For this lab:

| Field | Value |
|---|---|
| Source host | Kali Linux |
| Source IP | 192.168.64.4 |
| Destination host | Ubuntu Wazuh Server |
| Destination IP | 192.168.64.2 |
| Open ports | 22/tcp, 443/tcp |
| Expected services | SSH and HTTPS |

The activity was expected because I intentionally scanned my own lab server.

---

## Troubleshooting Notes

### Confirming the Correct Source Machine

At one point, a ping command was accidentally run from the Ubuntu server instead of Kali.

The terminal prompt showed:

```text
dustin@ubuntu
```

That indicated the command was being run from Ubuntu.

The correct Kali terminal prompt showed:

```text
kali@kali
```

After switching back to Kali, I re-ran the ping test and confirmed that Kali could reach the Wazuh server.

Lesson learned:

```text
Always verify the source system before documenting test results.
```

This matters because the same command can have a different meaning depending on which system it is run from.

---

### Service Fingerprint on Port 443

The Nmap service scan produced a long fingerprint for port 443.

This happened because Nmap detected that port 443 was open and responding over HTTPS, but it could not fully identify the exact application.

This was not a failed scan. The important finding was:

```text
443/tcp open ssl/https
```

In the context of this lab, HTTPS was expected because the Wazuh Dashboard uses HTTPS.

---

## Skills Demonstrated

This lab helped me practice:

- Kali Linux terminal usage
- Linux network interface review
- Basic connectivity testing
- Nmap scanning
- Service/version detection
- Comparing scan results from different source hosts
- Identifying expected open ports
- Basic attacker-versus-defender lab setup
- Safe scanning against authorized lab systems
- SOC-style network analysis

---

## Key Lessons Learned

- Kali can be used as a controlled attacker/testing machine in a home lab.
- `ip -br addr` shows Linux interface and IP information.
- `ping` confirms basic network reachability.
- Nmap identifies open ports and services.
- `nmap -sV` provides more detail than a basic scan.
- Open ports should be tied back to expected system roles.
- SSH on port 22 and HTTPS on port 443 were expected in this lab.
- Different source machines can be used to compare network visibility.
- Always verify which machine a command is being run from.
- Scanning should only be performed against systems I own or have permission to test.

---


## Next Steps

The next step is to generate controlled network activity from Kali and check whether the defender side can observe related events in Wazuh.

Future labs may include:

- SSH connection attempts from Kali
- Failed login monitoring
- Wazuh alert review
- Basic brute-force detection
- Mapping activity to MITRE ATT&CK
