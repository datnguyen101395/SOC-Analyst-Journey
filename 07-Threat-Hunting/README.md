# Lab 07 — SSH Login Monitoring with Wazuh

## Objective

The goal of this lab was to generate controlled failed SSH login activity from Kali Linux and verify that the activity was recorded on the Ubuntu server and detected in Wazuh.

This lab helped me understand how authentication activity moves from a Linux server log into a SIEM alert.

---

## Lab Environment

| Component | System |
|---|---|
| Attacker/testing machine | Kali Linux |
| Kali IP address | 192.168.64.4 |
| Target system | Ubuntu Wazuh Server |
| Target IP address | 192.168.64.2 |
| Monitored service | SSH |
| Destination port | 22/tcp |
| SIEM platform | Wazuh |
| Log source | `/var/log/auth.log` / journald |

---

## Lab Architecture

```text
Kali Linux
192.168.64.4
        ↓
Failed SSH login attempt
        ↓
Ubuntu Wazuh Server
192.168.64.2
        ↓
SSH authentication log
        ↓
Wazuh manager
        ↓
Wazuh SSH alert
        ↓
Analyst review
```

---

## Tasks Completed

During this lab, I completed the following tasks:

- Verified that SSH was available on the Ubuntu server.
- Confirmed that port 22 was listening.
- Generated a failed SSH login attempt from Kali.
- Used an invalid username to create controlled authentication failure activity.
- Reviewed Ubuntu authentication logs.
- Confirmed that Ubuntu recorded the failed SSH login.
- Found the SSH alert in Wazuh Threat Hunting.
- Expanded the Wazuh alert details.
- Reviewed source IP, username, decoder, rule ID, rule level, and full log.
- Documented the detection path from attacker activity to SIEM alert.

---

## SSH Service Verification

Before generating SSH login activity, I verified that SSH was available on the Ubuntu server.

The first command showed that `ssh.socket` was active:

```bash
systemctl is-active ssh.socket
```

I then confirmed that port 22 was listening:

```bash
sudo ss -tulnp | grep :22
```
<img width="856" height="135" alt="01-ubuntu-ssh-status" src="https://github.com/user-attachments/assets/59972073-e468-4e4f-a18a-71de9d2feb5a" />


The output showed:

```text
ssh.socket = active
port 22 = LISTENING
```

This confirmed that SSH was ready to accept connection attempts.

### Note on `ssh.service` and `ssh.socket`

When I first checked the SSH service, `ssh.service` showed as inactive. However, `ssh.socket` was active.

This means the system was using socket activation. In this configuration, systemd listens on port 22 and can start the SSH service when a connection attempt occurs.

Important lesson:

```text
ssh.service inactive does not always mean SSH is unavailable.

ssh.socket active + port 22 listening means SSH can accept connections.
```

---

## Failed SSH Login from Kali

From Kali Linux, I attempted to SSH into the Ubuntu server using an invalid username:

```bash
ssh fakeuser@192.168.64.2
```

I entered incorrect passwords until the login failed.

<img width="654" height="517" alt="02-kali-ssh-failed-login" src="https://github.com/user-attachments/assets/335cb1d2-874b-4aa5-bef3-9f45e5395412" />


The result showed:

```text
Permission denied (publickey,password).
```

This confirmed that Kali generated controlled failed SSH authentication activity against the Ubuntu server.

---

## Ubuntu Authentication Log Review

After generating the failed login attempt, I checked the Ubuntu authentication log for the attempted username.

Command used:

```bash
sudo grep "fakeuser" /var/log/auth.log
```
<img width="1151" height="172" alt="03-ubuntu-auth-log-failed-login" src="https://github.com/user-attachments/assets/a1d1f3b1-0c84-48e7-be17-5c241d4f0946" />



The log showed several important details:

```text
Invalid user fakeuser from 192.168.64.4
Failed password for invalid user fakeuser from 192.168.64.4
Connection closed by invalid user fakeuser 192.168.64.4 [preauth]
```

This confirmed that the Ubuntu server recorded the failed SSH activity locally.

---

## Wazuh SSH Alerts

After verifying the local Ubuntu logs, I searched Wazuh Threat Hunting for the failed SSH activity.

Search used:

```text
data.srcuser:"fakeuser"
```

<img width="3002" height="1420" alt="05-wazuh-ssh-alerts" src="https://github.com/user-attachments/assets/ff0536e4-2814-428e-9aa1-c91a846ecb1b" />

Wazuh showed multiple alerts related to the failed SSH login attempts.

The alert list showed:

| Field | Value |
|---|---|
| Agent | ubuntu |
| Rule description | `sshd: Attempt to login using a non-existent user` |
| Rule level | 5 |
| Rule ID | 5710 |
| Hits | 4 |

This confirmed that Wazuh collected the authentication log and generated SSH alerts.

---

## Expanded Wazuh Alert Details

I expanded one of the Wazuh SSH alerts to review the event fields.

<img width="1502" height="1528" alt="04-wazuh-ssh-expanded-alert" src="https://github.com/user-attachments/assets/092f940a-b226-4a84-894a-3f34303c379e" />


Important fields observed:

| Field | Value |
|---|---|
| `agent.name` | ubuntu |
| `data.srcip` | 192.168.64.4 |
| `data.srcuser` | fakeuser |
| `decoder.name` | sshd |
| `decoder.parent` | sshd |
| `location` | journald |
| `predecoder.program_name` | sshd-session |
| `rule.description` | sshd: Attempt to login using a non-existent user |
| `rule.id` | 5710 |
| `rule.level` | 5 |
| `rule.groups` | syslog, sshd, authentication_failed, invalid_login |

The `full_log` field showed the original log message:

```text
Failed password for invalid user fakeuser from 192.168.64.4
```

This confirmed that Wazuh parsed the failed SSH login and extracted useful investigation fields.

---

## Detection Path

This lab confirmed the full detection path:

```text
Kali SSH attempt
        ↓
Ubuntu SSH service receives login request
        ↓
Ubuntu rejects invalid username/password
        ↓
Ubuntu records event in authentication logs
        ↓
Wazuh collects the log
        ↓
Wazuh decodes it as SSH activity
        ↓
Wazuh rule 5710 triggers
        ↓
Alert appears in Threat Hunting
        ↓
Analyst reviews source IP, username, and rule details
```

---

## Analysis Summary

| Field | Value |
|---|---|
| Source host | Kali Linux |
| Source IP | 192.168.64.4 |
| Destination host | Ubuntu Wazuh Server |
| Destination IP | 192.168.64.2 |
| Destination port | 22/tcp |
| Protocol | SSH |
| Attempted username | fakeuser |
| Result | Failed login |
| Local log source | `/var/log/auth.log` |
| Wazuh decoder | sshd |
| Wazuh rule ID | 5710 |
| Wazuh rule level | 5 |
| Wazuh rule description | `sshd: Attempt to login using a non-existent user` |

---

## SOC Analyst Interpretation

From a SOC perspective, this alert indicates that a system attempted to authenticate to the Ubuntu server using a username that does not exist.

The key investigation questions are:

- What was the source IP?
- Is the source IP internal or external?
- Is the source host expected to connect to this server?
- What username was attempted?
- Does the username exist?
- How many failed attempts occurred?
- Did the attempts come from one source or many sources?
- Were there successful logins after the failed attempts?
- Is this part of normal admin activity, user error, scanning, or brute-force behavior?

In this lab, the activity was expected because it was generated intentionally from my Kali VM.

In a real environment, repeated failed SSH attempts from an unknown source would require further investigation.

---

## Troubleshooting Notes

### SSH Service Appeared Inactive

When I first checked SSH with:

```bash
sudo systemctl status ssh
```

The service showed:

```text
Active: inactive (dead)
TriggeredBy: ssh.socket
```

At first, this looked like SSH was not running.

After checking `ssh.socket` and port 22, I confirmed that SSH was still available through socket activation.

Commands used:

```bash
systemctl is-active ssh.socket
sudo ss -tulnp | grep :22
```

Lesson learned:

```text
A service can appear inactive while its socket is active and listening.
```

---

### Command Typo with `ss`

I accidentally typed:

```bash
sudo ss-tulnp
```

This failed because Linux interpreted `ss-tulnp` as one command.

The correct command was:

```bash
sudo ss -tulnp
```

Lesson learned:

```text
Linux commands require correct spacing between the command and its options.
```

---

### Verifying the Correct Source Machine

In the previous Kali lab, I accidentally ran a command from the Ubuntu terminal instead of the Kali terminal.

For this SSH lab, I paid attention to the terminal prompt:

```text
kali@kali
```

for the attacker machine, and:

```text
dustin@ubuntu
```

for the target/defender machine.

Lesson learned:

```text
Always verify which host the command is being run from before documenting results.
```

---

## Skills Demonstrated

This lab helped me practice:

- SSH service verification
- Linux socket/service troubleshooting
- Kali-to-Ubuntu SSH testing
- Controlled failed login generation
- Linux authentication log review
- Searching `/var/log/auth.log`
- Wazuh Threat Hunting search
- Wazuh alert analysis
- SSH decoder review
- Rule ID and rule level interpretation
- Source IP and username investigation
- Attacker-to-defender activity correlation
- Basic SOC authentication monitoring

---

## Key Lessons Learned

- SSH activity can be monitored through Linux authentication logs.
- Failed SSH logins are recorded in `/var/log/auth.log`.
- Wazuh can collect Linux authentication logs and generate alerts.
- Invalid usernames can trigger SSH-related Wazuh rules.
- `data.srcip` identifies the source IP of the login attempt.
- `data.srcuser` identifies the username used in the attempt.
- `rule.id` identifies the specific Wazuh detection rule.
- `rule.level` helps indicate alert severity.
- The `full_log` field preserves the original log message.
- A socket can listen for a service even when the service itself is not actively running.
- The source host, destination host, username, port, and result are key fields in an authentication investigation.

---


## Next Steps

The next step is to build on this lab by generating multiple failed SSH login attempts and reviewing how Wazuh handles repeated authentication failures.

Future improvements may include:

- Testing multiple failed SSH attempts.
- Reviewing brute-force related rules.
- Comparing invalid user attempts with wrong password attempts for a valid user.
- Creating an incident summary from the SSH alert.
- Mapping the activity to MITRE ATT&CK.
