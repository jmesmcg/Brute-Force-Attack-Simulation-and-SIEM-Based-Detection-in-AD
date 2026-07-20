# Brute-Force Attack Simulation and SIEM-Based Detection in AD

A home-lab SOC project that builds an Active Directory environment, simulates a real RDP brute-force attack against a domain-joined host, and detects it end-to-end using a self-hosted Splunk SIEM fed by Sysmon and Windows Event Logs.

## 📋 Table of Contents
- [Overview](#overview)
- [Features](#features)
- [Tools Used](#tools-used)
- [Lab Architecture](#lab-architecture)
- [Installation / Lab Setup](#installation--lab-setup)
- [Usage](#usage)
- [How It Works](#how-it-works)
- [Detection Reference](#detection-reference)
- [Output Example](#output-example)
- [Lessons Learned](#lessons-learned)
- [Next Steps](#next-steps)

## Overview

This project simulates a complete brute-force credential attack lifecycle, from environment build-out through attack execution to SIEM-based detection, inside an isolated Active Directory lab. A domain controller and domain-joined Windows 10 workstation were built in VirtualBox, endpoint telemetry (Windows Security Event Logs + Sysmon) was forwarded to a self-hosted Splunk indexer, and an RDP brute-force attack was launched from a Kali Linux attacker machine using Hydra. The resulting failed and successful logon events were then identified in Splunk to validate the compromise and document a repeatable detection methodology.

## Features

- **Full AD Environment** - Domain controller, domain-joined workstation, and forest built from scratch in VirtualBox
- **Centralized Log Forwarding** - Splunk Universal Forwarder on every Windows host, shipping to a single indexer
- **High-Fidelity Endpoint Telemetry** - Sysmon deployed with the Olaf Hartong `sysmon-modular` configuration
- **Real Attack Simulation** - RDP brute-force attack executed with Hydra against a live domain account
- **Custom Wordlist Engineering** - Password list built from a `rockyou.txt` sample plus known account passwords
- **SIEM Detection Logic** - Splunk searches correlating failed (`4625`) and successful (`4624`) logon events to confirm compromise
- **Source Attribution** - Attacker IP identified directly from Windows Security Event Log `Network Information` fields

## Tools Used

| Tool | Purpose |
|---|---|
| VirtualBox (NAT Network) | Virtualization / isolated lab network |
| Windows Server | Active Directory Domain Services / Domain Controller |
| Windows 10 | Domain-joined target workstation |
| Kali Linux | Attacker machine |
| Splunk Enterprise | Centralized SIEM / log indexer |
| Splunk Universal Forwarder | Log shipping from endpoints to Splunk |
| Sysmon (Olaf Hartong config) | High-fidelity endpoint telemetry |
| Hydra | RDP brute-force attack tool |
| rockyou.txt | Source wordlist for password candidates |


## Lab Architecture

**Network:** `192.168.10.0/24`

| Host | Role | IP Address |
|---|---|---|
| Splunk Server (Ubuntu) | SIEM / Log Indexer | `192.168.10.10` |
| Windows Server | Active Directory Domain Controller | `192.168.10.7` |
| Windows 10 | Domain-joined Target PC | `192.168.10.100` |
| Kali Linux | Attacker Machine | `192.168.10.250` |

## Installation / Lab Setup

```bash
# 1. Splunk Server - set static IP and enable receiving
sudo nano /etc/netplan/50-cloud-init.yaml   # set address to 192.168.10.10/24
sudo netplan apply
# In Splunk UI: Settings > Forwarding and receiving > Configure receiving > port 9997

# 2. Domain Controller - promote to AD DS, create forest, add domain users
# (Server Manager > Add Roles and Features > AD DS > Promote to DC)

# 3. Target PC - join domain, install Splunk UF + Sysmon
# Copy inputs.conf from etc/system/default to etc/system/local, forward Security/Sysmon logs
Sysmon64.exe -i sysmonconfig.xml

# 4. Kali (Attacker) - build password list
cd /usr/share/wordlists
sudo gunzip rockyou.txt.gz
head -n 20 rockyou.txt > passwords.txt
# append known/candidate account passwords to passwords.txt
```

## Usage

```bash
# Run the brute-force attack against the target's RDP service
hydra -l <username> -P passwords.txt rdp://192.168.10.100 -t 1 -W 2

# In Splunk - search for failed logon attempts
index="endpoint" <username> EventCode=4625

# In Splunk - confirm a successful logon followed the failures
index="endpoint" <username> EventCode=4624
```

## How It Works

```
Kali (Attacker)                Target PC                    Splunk Server
      │                             │                              │
      │  Hydra RDP brute-force      │                              │
      ├────────────────────────────►│                              │
      │  (iterates passwords.txt)   │                              │
      │                             │  Sysmon + Windows Security   │
      │                             │  logs forwarded via UF       │
      │                             ├─────────────────────────────►│
      │                             │                              │
      │      Valid credential       │                              │
      │◄─────────────────found──────┤                              │
      │                             │                              │  Analyst searches:
      │                             │                              │  EventCode=4625 (failed)
      │                             │                              │  EventCode=4624 (success)
      │                             │                              │  → attack confirmed
```

## Detection Reference

| Event ID | Description | Analyst Significance |
|---|---|---|
| `4625` | An account failed to log on | High-frequency repeats from one source IP = brute-force indicator |
| `4624` | An account was successfully logged on | Confirms compromise; anchor point for tracing post-logon activity |

**Detection Logic:** A high volume of `4625` events for a single account within a short time window, followed immediately by a `4624` for that same account, is a reliable signature that a brute-force attempt succeeded - and is easily operationalized as a scheduled Splunk correlation search / alert.

## Output Example



## Lessons Learned

- **Windows Event Log fundamentals** - how logon events, security IDs, and network information fields tie an authentication event back to a specific source
- **SIEM log correlation** - pairing `4625` and `4624` events by account and time window turns raw logs into an actual detection
- **Sysmon configuration** - deploying a community-hardened config (Olaf Hartong) instead of default logging dramatically increases telemetry fidelity
- **Attacker-side tooling** - how Crowbar structures RDP brute-force attempts and why targeted wordlists (not just raw rockyou.txt) matter for realistic testing
- **Splunk forwarder architecture** - how `inputs.conf` on the Universal Forwarder controls exactly what telemetry reaches the indexer

## Next Steps

- [ ] Build a scheduled Splunk correlation search / alert for the 4625→4624 pattern
- [ ] Add account lockout policy testing to see how it changes the detection window
- [ ] Extend telemetry to include process creation (Event ID 4688) for post-compromise activity
- [ ] Add MFA to RDP and re-run the attack to validate the control
- [ ] Build a Splunk dashboard visualizing failed-logon volume by account/source over time

---
