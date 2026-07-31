---
title: "Building a Home SOC with Wazuh – Part 1"
description: "A hands-on walkthrough of building a three-machine Wazuh SOC lab and using it to detect Nmap reconnaissance and SSH brute-force attacks through real log analysis, MITRE ATT&CK mapping, and analyst-style investigation."
date: 2026-07-31
author: "abdrx"
category: "Project"
series: "Home SOC"
part: 1
tags:
  - Wazuh
  - SOC
  - SIEM
difficulty: Intermediate
readingTime: 10 min
cover: banner.webp
featured: true
repository: "https://github.com/madebyabder/SOC-Simulation-Wazuh"
medium: "https://medium.com/@madebyabder/building-a-soc-from-scratch-detecting-reconnaissance-and-ssh-brute-force-with-wazuh-part-1-611304a4e08c?sharedUserId=madebyabder"
---

# Building a SOC From Scratch: Detecting Reconnaissance and SSH Brute-Force with Wazuh (Part 1)

## Overview

Installing a SIEM is only the first step. The real value comes from understanding the telemetry it collects, recognizing malicious behavior, and investigating alerts the way a SOC analyst would.

To practice those skills, I built a small Security Operations Center (SOC) lab using Wazuh inside an isolated virtual environment. Rather than simply deploying the platform, I simulated a realistic attack chain and analyzed how each stage appeared inside the SIEM.

This article is the first part of a two-part series documenting that project. In this article I cover:

- Building the lab architecture
- Preparing the monitored Linux server
- Validating telemetry collection
- Detecting reconnaissance using Nmap
- Detecting SSH brute-force attempts using Hydra
- Evaluating alert severity and possible false positives

The second article continues with web application attacks, suspicious command execution, persistence techniques, simulated data exfiltration, and attack-chain correlation.

## Why I Built This

Reading about SIEM platforms and detection rules is useful, but actually seeing your own attack traffic become alerts teaches much more. Wazuh is a good fit for this because it's free and open-source, and it exposes the pieces that actually matter for learning: raw log collection, a rule engine, MITRE ATT&CK mapping, and File Integrity Monitoring (FIM).

## Objectives

My objective wasn't simply to install Wazuh. It was to understand the complete detection workflow:

- Generate realistic attacker activity
- Verify telemetry reaches the SIEM
- Investigate the resulting alerts
- Map detections to MITRE ATT&CK
- Assess alert severity
- Document the investigation the way a SOC analyst would

## Lab Architecture

The environment consists of three virtual machines connected through a VirtualBox Host-Only network. This configuration isolates every attack from the public internet while allowing communication between the attacker, the monitored endpoint, and the SOC server.

- **Kali Linux** — Attacker machine — `192.168.56.10`
- **Ubuntu Server** — Monitored endpoint — `192.168.56.20`
- **Wazuh Server** — SIEM platform — `192.168.56.30`

Using a Host-Only network keeps every simulated attack contained to machines I own, and it forces the same kind of network segmentation thinking you'd apply in a real environment, where a monitored segment shouldn't have arbitrary internet access.

## Technologies Used

- **Wazuh** — SIEM platform (rule engine, MITRE ATT&CK mapping, FIM)
- **Kali Linux** — attacker machine
- **Ubuntu Server** — monitored endpoint
- **OpenSSH** — remote access service under test
- **Nginx** — web server generating HTTP telemetry
- **Nmap** — reconnaissance / port and service scanning
- **THC Hydra** — SSH brute-force / password-guessing simulation
- **VirtualBox (Host-Only networking)** — isolated lab environment

## Environment

Ubuntu Server was configured to run two services that generate useful security logs: OpenSSH and Nginx.

The two log sources that matter for this whole project come from these services:

- `/var/log/auth.log` — SSH logins, failures, invalid users, sudo activity
- `/var/log/nginx/access.log` — every HTTP request hitting the box

Before launching any attacks, I confirmed that logs generated on Ubuntu were actually reaching Wazuh. To verify the pipeline, I generated three simple events: a successful SSH login, a failed SSH login, and a sudo command execution. After a few moments, each event appeared inside the Wazuh Dashboard with useful investigation fields, including `agent.name`, `agent.ip`, `data.srcip`, `data.srcuser`, `rule.description`, and `rule.level`.

This step is often skipped in tutorials, but it's essential. If telemetry isn't working correctly, later investigations become unreliable, because it's impossible to tell whether an attack wasn't detected or whether the logging pipeline simply failed.

## Installation

```bash
sudo apt update
sudo apt install openssh-server nginx curl net-tools -y
sudo systemctl enable ssh --now
sudo systemctl enable nginx --now
```

## Configuration

### Detecting Reconnaissance

Reconnaissance is usually the first observable stage of an intrusion. Before attempting exploitation, attackers often identify open ports and running services. To simulate this, I ran a basic scan and a service-version scan from Kali against the monitored Ubuntu server.

Instead of immediately producing SSH-related alerts, the scans generated HTTP requests against the Nginx web server. Inside the access logs, I observed requests to paths such as `/HNAP1` and `/sdk`, common artifacts produced by automated scanning tools.

Inside Wazuh, I filtered events using:

```
data.srcip: "192.168.56.10"
nginx
```

The alert data included:

- `agent.ip`: 192.168.56.20
- `data.srcip`: 192.168.56.10
- `data.url`: /HNAP1, /sdk
- `rule.description`: Web server 400 error code
- `rule.level`: 5

One interesting lesson from this stage was that reconnaissance doesn't always appear where you expect. Since the scan interacted primarily with the web server, the most valuable evidence came from Nginx logs rather than SSH logs — no `sshd` events were generated at all. This reinforced why relying on a single log source is risky: a SOC watching only SSH would have missed this scan entirely. Firewall logs, an IDS like Suricata, or network flow data would round out visibility here.

**MITRE ATT&CK techniques observed:** T1595 (Active Scanning), T1046 (Network Service Discovery)

### Detecting SSH Brute-Force

After validating reconnaissance detection, I simulated an automated SSH password-guessing attack using Hydra. The objective wasn't to compromise the server, but to generate realistic authentication failures for investigation.

```bash
hydra -L users.txt -P passwords.txt ssh://192.168.56.20 -t 2 -V
```

Hydra performed 25 authentication attempts. No valid credentials were discovered, which was expected — the focus of this exercise was detection, not compromise. Ubuntu's authentication log showed failed passwords, invalid users, PAM authentication failures, and SSH disconnect events.

Inside Wazuh, I searched for:

```
sshd
data.srcip: "192.168.56.10"
```

The alerts included:

- `data.srcuser`: fakeuser / test / labuser
- `rule.description`: sshd: Attempt to login using a non-existent user
- `rule.description`: syslog: User missed the password more than one time
- `rule.level`: 5 / 10
- `rule.mitre.id`: T1110
- `rule.mitre.technique`: Password Guessing
- `rule.mitre.tactic`: Credential Access

### Thinking Like a SOC Analyst

A single failed SSH login rarely indicates malicious activity. What made this event suspicious was the combination of several indicators: multiple authentication failures, multiple usernames including nonexistent ones, repeated attempts from the same IP address, a short time interval between attempts, and increasing alert severity as attempts repeated.

Based on those indicators, I classified this event as **Medium-to-High severity** — high enough to warrant investigation because it's clearly automated, but not Critical, since no login actually succeeded. It would escalate to High or Critical if a successful login followed the failures, or if similar activity spread across multiple systems. That escalation logic is something I'll come back to in Part 2, once this brute-force attempt gets correlated with everything that follows it.

## Challenges

**Considering false positives:** Before escalating an alert, it's worth ruling out legitimate explanations:

- A user entering the wrong password several times
- An administrator testing multiple accounts
- Automation using outdated credentials
- A service account with an expired password

Correlating multiple indicators — number of usernames, timing, and source IP — is what separates a real brute-force pattern from ordinary noise.

**Best practices / mitigation:**

*Reconnaissance:*
- Limit unnecessary exposed services
- Use host firewalls to log and restrict inbound traffic
- Deploy a lightweight IDS such as Suricata for additional visibility, since endpoint logs alone may miss reconnaissance that never touches a monitored service

*SSH Brute Force:*
- Prefer SSH key authentication over passwords
- Deploy Fail2Ban or a similar rate-limiting control
- Restrict SSH access to trusted networks where possible
- Alert on the same correlation signals used above: multiple usernames, one source IP, short time window

## Lessons Learned

- Always validate your telemetry before testing detections. A missing alert may mean a broken logging pipeline, not an undetected attack.
- Reconnaissance activity may appear in unexpected log sources depending on which service it actually touches.
- A single failed login is noise. Repetition, multiple usernames, a consistent source IP, and a short time window are what turn noise into a credible brute-force indicator.
- Effective SOC analysis depends on context. Individual alerts rarely tell the full story, but correlating multiple observations is what lets an analyst separate genuinely suspicious behavior from false positives.

## What's Next

Building the environment was only the beginning. The real learning came from validating telemetry, investigating alerts, understanding why Wazuh generated each detection, and documenting the investigation from a defender's perspective.

Reconnaissance and SSH brute-force are relatively simple attack techniques, but they already cover many of the core skills expected from a SOC analyst: log analysis, alert triage, MITRE ATT&CK mapping, severity assessment, and structured investigation.

In **Part 2**, I'll continue the attack chain with web application attacks, suspicious command execution, persistence techniques, simulated data exfiltration, and full attack-chain correlation.

## References

- MITRE ATT&CK Framework
- Wazuh Documentation
- Nmap Documentation
- THC Hydra Documentation
