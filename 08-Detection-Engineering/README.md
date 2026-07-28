# Lab 08 — Detection Engineering

## Overview

This lab focuses on understanding how security telemetry becomes a detection in Wazuh.

For the first exercise, I validated Wazuh's built-in SSH detection logic by generating repeated failed SSH login attempts from Kali Linux and reviewing how the activity appeared in Ubuntu logs and Wazuh alerts.

This lab did not create a custom detection rule yet. Instead, it focused on detection validation, alert analysis, and understanding how existing Wazuh rules process authentication failures.

## Exercises

| Exercise | Description |
|---|---|
| SSH Failed Login Analysis | Generated repeated failed SSH login attempts from Kali, verified Ubuntu authentication logs, and reviewed Wazuh SSH alerts. |

## Current Detection Focus

The current exercise uses Wazuh's built-in SSH rule:

| Field | Value |
|---|---|
| Rule ID | 5710 |
| Rule Description | `sshd: Attempt to login using a non-existent user` |
| Rule Level | 5 |
| Decoder | sshd |
| Source IP | 192.168.64.4 |
| Attempted Username | fakeuser |

## Detection Path

```text
Kali SSH attempts
        ↓
Ubuntu authentication logs
        ↓
Wazuh decoder: sshd
        ↓
Wazuh rule 5710
        ↓
Alert review
