# Lab 08 — Repeated SSH Failed Login Detection with Wazuh

## Objective

The goal of this lab was to generate repeated failed SSH login attempts from Kali Linux against an Ubuntu Wazuh server and investigate how those attempts appeared in Linux authentication logs and Wazuh.

This lab builds on Lab 07, where I tested a single failed SSH login. In this lab, I repeated the failed login activity to create a pattern that could be reviewed from a SOC analyst perspective.

Important note:

```text
Wazuh generated multiple failed SSH login alerts.
In this lab run, Wazuh did not trigger a separate brute-force escalation rule.
The detection was still useful because the alerts showed repeated failed login attempts from the same source IP and username.
```

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
| Wazuh decoder | sshd |

---

## Lab Architecture

```text
Kali Linux
192.168.64.4
        ↓
Repeated failed SSH login attempts
        ↓
Ubuntu Wazuh Server
192.168.64.2
        ↓
SSH authentication logs
        ↓
Wazuh manager
        ↓
Repeated SSH failed-login alerts
        ↓
Analyst review
```

---

## Tasks Completed

During this lab, I completed the following tasks:

- Generated repeated failed SSH login attempts from Kali.
- Used an invalid username to create controlled failed authentication activity.
- Verified that Ubuntu recorded the failed login attempts in `/var/log/auth.log`.
- Searched Wazuh Threat Hunting for the attempted username.
- Confirmed multiple Wazuh alerts were generated.
- Expanded a Wazuh SSH alert and reviewed important fields.
- Identified the source IP, attempted username, decoder, rule ID, rule level, and full log.
- Reviewed the event timeline to confirm repeated activity in a short time window.
- Documented the difference between repeated failed login alerts and a separate brute-force escalation rule.

---

## Step 1 — Repeated SSH Failures from Kali

From Kali Linux, I attempted to SSH into the Ubuntu Wazuh server using an invalid username:

```bash
ssh fakeuser@192.168.64.2
```

I entered incorrect passwords until the login failed.

<img width="649" height="515" alt="01-Kali-repeated-ssh-failures" src="https://github.com/user-attachments/assets/3e2a493c-ef38-4e29-a752-3f60364df222" />


The SSH client returned:

```text
Permission denied, please try again.
Permission denied (publickey,password).
```

This confirmed that Kali generated failed SSH login attempts against the Ubuntu server.

---

## Step 2 — Ubuntu Authentication Log Review

After generating the failed SSH attempts, I reviewed the Ubuntu authentication log.

Command used:

```bash
sudo grep "fakeuser" /var/log/auth.log | tail -30
```

<img width="1211" height="243" alt="02-ubuntu-auth-log-multiple-failures" src="https://github.com/user-attachments/assets/20e65115-7a90-48f5-bd05-11a0dfd57833" />


The log showed repeated SSH authentication failures, including:

```text
Invalid user fakeuser from 192.168.64.4
Failed password for invalid user fakeuser from 192.168.64.4
Connection closed by invalid user fakeuser 192.168.64.4 [preauth]
```

This confirmed that the Ubuntu server recorded the failed login attempts locally before the events were reviewed in Wazuh.

---

## Step 3 — Wazuh SSH Alerts

In Wazuh Threat Hunting, I searched for the attempted username:

```text
data.srcuser:"fakeuser"
```

<img width="1448" height="782" alt="03-wazuh-ssh-bruteforce-alerts" src="https://github.com/user-attachments/assets/dc02faeb-b8b8-4fa6-9b47-584cb957fbec" />


Wazuh returned multiple SSH authentication failure alerts.

Observed alert details:

| Field | Value |
|---|---|
| Agent | ubuntu |
| Source user | fakeuser |
| Rule description | `sshd: Attempt to login using a non-existent user` |
| Rule ID | 5710 |
| Rule level | 5 |
| Number of hits | 4 |

This confirmed that Wazuh collected the SSH authentication logs and generated alerts for the repeated failed login attempts.

---

## Step 4 — Expanded Alert Review

I expanded one of the Wazuh SSH alerts to review the parsed fields.

<img width="1448" height="782" alt="04-expanded-bruteforce-alert-details" src="https://github.com/user-attachments/assets/67b6376f-bd58-4c5b-bfad-be354c6f4b20" />


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
| `rule.description` | `sshd: Attempt to login using a non-existent user` |
| `rule.id` | 5710 |
| `rule.level` | 5 |
| `rule.firedtimes` | 4 |
| `rule.groups` | syslog, sshd, authentication_failed, invalid_login |

The `full_log` field showed the original event:

```text
Failed password for invalid user fakeuser from 192.168.64.4
```

The `rule.firedtimes` value showed that the rule fired multiple times for this activity.

---

## Step 5 — SSH Attack Timeline

I reviewed the alert timeline in Wazuh to understand when the repeated failed login activity occurred.

<img width="1441" height="444" alt="05-ssh-attack-timeline" src="https://github.com/user-attachments/assets/fdbbe78b-2c66-49f7-b39a-31293043162f" />


The timeline showed multiple alerts grouped in the same short time window.

This supported the conclusion that the activity was not a single mistake, but repeated failed login behavior from the same source.

---

## Detection Path

This lab confirmed the following detection path:

```text
Kali repeated SSH login attempts
        ↓
Ubuntu SSH service receives the attempts
        ↓
Ubuntu rejects the invalid username/password
        ↓
Ubuntu records the failures in authentication logs
        ↓
Wazuh collects the logs
        ↓
Wazuh decodes the events as SSH activity
        ↓
Wazuh rule 5710 triggers multiple times
        ↓
Alerts appear in Threat Hunting
        ↓
Analyst reviews source IP, username, rule details, and timeline
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
| Result | Repeated failed login attempts |
| Local log source | `/var/log/auth.log` |
| Wazuh decoder | sshd |
| Wazuh rule ID | 5710 |
| Wazuh rule level | 5 |
| Wazuh rule fired times | 4 |
| Wazuh rule description | `sshd: Attempt to login using a non-existent user` |

---

## SOC Analyst Interpretation

From a SOC perspective, repeated failed SSH login attempts can indicate:

- User error
- Misconfigured automation
- Password guessing
- Reconnaissance
- Brute-force activity
- Unauthorized access attempts

In this lab, the activity was expected because I generated it intentionally from my Kali VM.

In a real environment, I would investigate:

- Whether the source IP is internal or external.
- Whether the source host is expected to connect to the server.
- Whether the attempted username exists.
- How many failed attempts occurred.
- Whether the attempts targeted multiple usernames.
- Whether there was a successful login after the failures.
- Whether other hosts received similar login attempts.
- Whether the source IP should be blocked or escalated.

---

## Important Finding

This lab did not show a separate high-severity brute-force escalation rule.

Instead, Wazuh showed multiple SSH invalid-login alerts using rule ID 5710.

That is still valuable because repeated rule 5710 alerts from the same source IP and username can indicate suspicious authentication behavior.

Accurate conclusion:

```text
Wazuh detected repeated failed SSH login attempts from Kali.
The activity appeared as multiple SSH invalid-user alerts.
The alert details showed source IP, username, rule ID, rule level, full log, and rule fired count.
```

---

## Troubleshooting Notes

### No Separate Brute-Force Rule Triggered

I expected that repeated failed SSH attempts might trigger a separate brute-force alert.

In this lab run, Wazuh generated multiple rule 5710 alerts instead.

This showed me that repeated activity does not always automatically escalate to a different rule unless the rule conditions, frequency, timeframe, and correlation requirements are met.

Lesson learned:

```text
Multiple failed login alerts can still indicate a brute-force pattern, even if a separate brute-force rule does not trigger.
```

---

### Sudo Activity Also Appeared in Auth Logs

When searching `/var/log/auth.log`, the log also showed my `sudo grep` command.

This happened because Ubuntu records sudo activity in the authentication log.

That was expected behavior.

The SSH-related lines were the evidence used for this lab.

---

### Importance of Time Windows

In Wazuh, the time range affected which alerts appeared.

To find the activity, I used a recent time window and searched for:

```text
data.srcuser:"fakeuser"
```

Lesson learned:

```text
When searching SIEM alerts, verify the time range and filters before assuming events are missing.
```

---

## Skills Demonstrated

This lab helped me practice:

- Generating controlled failed SSH login attempts
- Reviewing Linux authentication logs
- Searching logs with `grep`
- Using `tail` to review recent log entries
- Searching Wazuh Threat Hunting
- Filtering alerts by username
- Reviewing Wazuh SSH alert details
- Understanding Wazuh rule IDs and rule levels
- Interpreting repeated failed authentication attempts
- Reviewing alert timelines
- Differentiating repeated alerts from a separate brute-force escalation rule
- SOC-style authentication investigation

---

## Key Lessons Learned

- Repeated failed SSH logins are recorded in Ubuntu authentication logs.
- Wazuh can detect SSH login attempts using the `sshd` decoder.
- Rule ID 5710 identifies attempts to log in with a non-existent user.
- `data.srcip` shows where the login attempt came from.
- `data.srcuser` shows the attempted username.
- `full_log` preserves the original Linux log message.
- `rule.firedtimes` can help show repeated rule matches.
- Multiple failed login alerts can indicate suspicious behavior.
- A separate brute-force escalation rule may not always trigger automatically.
- Time range and filters are important when searching SIEM events.
- Repeated authentication failures should be reviewed for source, target, username, count, and time window.

---

## Next Steps

Future improvements for this lab could include:

- Testing more failed SSH attempts.
- Comparing invalid-user attempts with wrong passwords for a valid user.
- Reviewing Wazuh frequency-based SSH rules.
- Generating a higher-confidence brute-force alert.
- Writing a short incident report based on the alert.
- Mapping the activity to MITRE ATT&CK.
