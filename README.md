# Wazuh SOC Home Lab

A small home lab I built to practice the core SOC analyst workflow: collect logs from different systems, generate realistic attack activity in a controlled environment, and investigate the resulting alerts in Wazuh.

The goal is not to build a production environment. It is to get hands-on experience with log collection, alert triage, detection engineering, and mapping detections to attacker behavior.


---

## Lab Architecture

The lab runs as four virtual machines on a single VMware Workstation host, connected through a host-only network.

<img width="1880" height="940" alt="wazuh-lab-architecture" src="https://github.com/user-attachments/assets/6135b8ab-1f98-469c-9795-4c44ad557f39" />

*Architecture screenshot of the lab (VMs, IP addresses, and how they connect).*

```text
┌─────────────────────────────────────────────────────┐
│             VMware Host-Only Network                │
│                 192.168.160.0/24                    │
│                                                     │
│  ┌───────────────┐  ┌──────────────────────────┐    │
│  │  Kali Linux   │  │     Wazuh Manager        │    │
│  │192.168.160.131│──│     Ubuntu 24.04 LTS     │    │
│  │   Attack VM   │  │     192.168.160.128       │    │
│  └───────────────┘  │     Wazuh 4.14.4         │    │
│                     └──────────┬───────────────┘    │
│                                │                    │
│                  ┌─────────────┴─────────────┐      │
│                  │                           │      │
│  ┌───────────────┴─────────┐  ┌──────────────┴──┐  │
│  │       Windows 10        │  │ Ubuntu 24.04 LTS │  │
│  │     192.168.160.130     │  │ 192.168.160.129  │  │
│  │     Wazuh Agent 001     │  │ Wazuh Agent 002  │  │
│  │       Sysmon v15.15     │  │     auditd       │  │
│  └──────────────────────────┘  └─────────────────┘  │
└─────────────────────────────────────────────────────┘



```

### Lab components

| VM | OS | IP | Role |
|---|---|---|---|
| Attack VM | Kali Linux | `192.168.160.131` | Generates controlled malicious activity against the endpoints |
| Wazuh Manager | Ubuntu 24.04 LTS | `192.168.160.128` | Wazuh Manager and Dashboard; receives agent data, applies detection rules, and generates alerts |
| Windows endpoint | Windows 10 | `192.168.160.130` | Wazuh Agent 001; Windows Event Logs and Sysmon telemetry |
| Ubuntu endpoint | Ubuntu 24.04 LTS | `192.168.160.129` | Wazuh Agent 002; authentication, sudo, system logs, auditd and FIM telemetry |

### How the machines interact

1. The Wazuh agents on both endpoints collect local logs and events and send them to the Wazuh Manager.
2. The manager analyzes the incoming data against its rules and raises alerts.
3. The Kali machine runs controlled attack simulations against the endpoints.
4. I review the resulting alerts in the Wazuh Dashboard and validate whether the activity was detected correctly.

---

## Tools

- **Wazuh 4.14.4** — log collection, detection and alerting
- **VMware Workstation** — virtualization and host-only networking
- **Ubuntu Server 24.04 LTS** — Wazuh Manager and Dashboard
- **Ubuntu 24.04 LTS** — monitored Linux endpoint
- **Windows 10** — monitored Windows endpoint
- **Sysmon v15.15** — detailed Windows process and system telemetry
- **auditd** — Linux command and system call auditing
- **Kali Linux** — controlled attack simulation

---

## Environment

| Item | Details |
|---|---|
| Hypervisor | VMware Workstation |
| Wazuh version | 4.14.4 |
| Ubuntu Server version | Ubuntu 24.04 LTS |
| Ubuntu Endpoint version | Ubuntu 24.04 LTS |
| Windows Endpoint version | Windows 10 |
| Attacker OS | Kali Linux |
| Network setup | VMware host-only network, `192.168.160.0/24` |

---

# Detection Use Cases

### Status key

- ✅ **Implemented and tested:** Activity was executed and the expected custom alert was confirmed in Wazuh.
- 🔧 **In progress:** Rules have been written but still require testing or tuning.
- 📝 **Planned:** Not implemented yet.

The custom detection rules are organized under the `rules/` directory.

### Detection overview

| # | Use case | What is detected | Data source | Custom rules | Status |
|---|---|---|---|---|---|
| 1 | SSH brute force and failed logins | Repeated failed SSH logins, invalid and valid users, root login over SSH | Linux authentication logs | 100502, 100503, 100504 | ✅ Implemented |
| 2 | Privilege escalation and suspicious sudo | Sudo to root and privileged Windows group changes | Linux auth logs, Windows Security logs | 100501, 100402 | ✅ Implemented |
| 3 | Account and group changes | Windows account changes, Linux user changes, critical authentication file modifications | Windows Security logs, Linux logs, FIM | 100401, 100402, 100505, 100510 | ✅ Implemented |
| 4 | Suspicious PowerShell and process execution | Suspicious PowerShell commands and curl/wget execution | Sysmon, auditd | 100403, 100509 | ✅ Implemented |
| 5 | Persistence and scheduled tasks | Windows scheduled tasks/services and Linux cron, systemd and SSH key changes | Windows Security/System logs, FIM | 100404, 100405, 100506, 100507, 100508 | ✅ Implemented |
| 6 | Network scanning and suspicious connections | Port scans and unusual outbound connections | Network telemetry | None yet | 📝 Planned |

### Network scanning note

Wazuh agents primarily collect host telemetry. A port scan from Kali against an endpoint is not directly visible to the Wazuh agents alone.

Use case 6 therefore requires an additional network telemetry source such as firewall logging or Suricata before dedicated detection rules can be implemented.

---

# Telemetry Required

| Where | What must be enabled |
|---|---|
| Windows agent | Sysmon process creation with command line |
| Windows agent | Windows Security auditing for account and group management |
| Windows agent | Event collection for Security, System and Sysmon |
| Ubuntu agent | FIM for `/etc/cron*`, `/var/spool/cron`, `/etc/systemd/system`, `/etc/passwd`, `/etc/shadow`, `/etc/sudoers` and `.ssh/authorized_keys` |
| Ubuntu agent | auditd with `execve` monitoring for `curl` and `wget` |

---

# Custom Detection Rules

The custom rules use Wazuh rule IDs from `100401` to `100510`.

| ID | Level | Detects | ATT&CK |
|---|---:|---|---|
| 100401 | 10 | Windows account created, enabled, password reset, disabled or deleted | T1136.001, T1098, T1531 |
| 100402 | 12 | Privileged group membership changes | T1098, T1078 |
| 100403 | 12 | Suspicious PowerShell execution involving download or execution techniques | T1059.001, T1027, T1105 |
| 100404 | 10 | Windows scheduled task creation | T1053.005 |
| 100405 | 10 | New Windows service installation | T1543.003 |
| 100501 | 10 | Successful sudo to root | T1548.003 |
| 100502 | 10 | Repeated SSH attempts using invalid users | T1110 |
| 100503 | 10 | Repeated failed SSH passwords for valid users | T1110 |
| 100504 | 12 | Successful SSH login as root | T1078 |
| 100505 | 10 | Linux user account creation or modification | T1136.001 |
| 100506 | 10 | Cron persistence file modification | T1053.003 |
| 100507 | 10 | SSH `authorized_keys` modification | T1098.004 |
| 100508 | 10 | Systemd persistence file modification | T1543.002 |
| 100509 | 10 | `curl` or `wget` execution | T1105 |
| 100510 | 12 | Modification of `/etc/passwd`, `/etc/shadow` or `/etc/sudoers` | T1098, T1548.003 |

Every custom rule contains MITRE ATT&CK mappings that can be used when investigating the resulting alerts in Wazuh.

---

# Detection Rule Organization

Rules are separated by operating system:

```text
rules/
├── windows/
│   ├── 100401_windows_account_modification.xml
│   ├── 100402_windows_privileged_group.xml
│   ├── 100403_windows_suspicious_powershell.xml
│   ├── 100404_windows_scheduled_task.xml
│   └── 100405_windows_service.xml
│
└── linux/
    ├── 100501_linux_sudo_privilege_escalation.xml
    ├── 100502_linux_ssh_invalid_user_bruteforce.xml
    ├── 100503_linux_ssh_failed_password_bruteforce.xml
    ├── 100504_linux_root_ssh_login.xml
    ├── 100505_linux_user_account_modification.xml
    ├── 100506_linux_cron_persistence.xml
    ├── 100507_linux_authorized_keys.xml
    ├── 100508_linux_systemd_persistence.xml
    ├── 100509_linux_curl_wget_execution.xml
    └── 100510_linux_critical_auth_files.xml
```

This makes each rule independently maintainable and easier to troubleshoot.

---

# Results Per Use Case

Each detection was tested by running controlled activity against the relevant endpoint and checking the resulting alert in the Wazuh Dashboard.

## 1. SSH Brute Force

**Simulation:** repeated failed SSH authentication attempts against the Ubuntu endpoint.

**Custom rules triggered:**

- `100502` — invalid-user brute force
- `100503` — failed-password brute force
- `100504` — root SSH login

**Result:** ✅ Confirmed in Wazuh

---

## 2. Privilege Escalation

**Simulation:** `sudo` privilege escalation on Ubuntu and privileged Windows group membership modification.

**Custom rules triggered:**

- `100501` — Linux sudo privilege escalation
- `100402` — Windows privileged group membership change

**Result:** ✅ Confirmed in Wazuh

---

## 3. Account and Group Changes

**Simulation:** Windows account changes, privileged group membership changes, Linux user changes and modification of critical Linux authentication files.

**Custom rules triggered:**

- `100401`
- `100402`
- `100505`
- `100510`

**Result:** ✅ Confirmed in Wazuh

Rule `100510` was validated by modifying the metadata of `/etc/passwd` using `touch`, producing the expected alert:

```text
Critical Linux authentication file modified: /etc/passwd
```

---

## 4. Suspicious PowerShell and Process Execution

**Simulation:** suspicious PowerShell download/command execution on Windows and `curl` execution on Ubuntu.

**Custom rules triggered:**

- `100403` — suspicious PowerShell
- `100509` — curl/wget execution

**Result:** ✅ Confirmed in Wazuh

---

## 5. Persistence and Scheduled Tasks

**Simulation:** Windows scheduled task and service creation; Linux cron, `authorized_keys` and systemd file changes.

**Custom rules triggered:**

- `100404` — Windows scheduled task
- `100405` — Windows service
- `100506` — Linux cron persistence
- `100507` — SSH authorized_keys persistence
- `100508` — systemd persistence

**Result:** ✅ Confirmed in Wazuh

---

## 6. Network Scanning

**Custom rules:** None yet.

**Result:** 📝 Planned

A network telemetry source such as firewall logging or Suricata is required before implementing this use case.

---

# MITRE ATT&CK Coverage

| Use case | ATT&CK technique | Rules | Detection |
|---|---|---|---|
| SSH brute force | T1110 — Brute Force | 100502, 100503 | Repeated failed SSH authentication |
| SSH authentication | T1078 — Valid Accounts | 100504 | Root SSH login |
| Privilege escalation | T1548.003 — Sudo and Sudo Caching | 100501, 100510 | Sudo activity and sudoers modification |
| Privilege escalation | T1078 — Valid Accounts | 100402 | Privileged Windows group membership |
| Account changes | T1136.001 — Create Account: Local Account | 100401, 100505 | Local account creation |
| Account manipulation | T1098 — Account Manipulation | 100401, 100402, 100510 | Account and privilege changes |
| Account access removal | T1531 — Account Access Removal | 100401 | Account disabling/deletion |
| PowerShell | T1059.001 — PowerShell | 100403 | Suspicious PowerShell execution |
| Tool transfer | T1105 — Ingress Tool Transfer | 100509 | curl/wget execution |
| Scheduled task | T1053.005 — Scheduled Task | 100404 | Windows scheduled task creation |
| Cron | T1053.003 — Cron | 100506 | Linux cron modification |
| Windows service | T1543.003 — Windows Service | 100405 | New Windows service |
| Systemd service | T1543.002 — Systemd Service | 100508 | Systemd unit modification |
| SSH authorized keys | T1098.004 — SSH Authorized Keys | 100507 | `authorized_keys` modification |
| Network discovery | T1046 — Network Service Discovery | None yet | Planned |

---

# What I'm Practicing

Through this lab, I am practicing:

- Deploying and connecting a Wazuh Manager and agents
- Configuring Windows and Linux telemetry sources
- Writing and tuning custom Wazuh detection rules
- Reading and triaging security alerts
- Separating useful signals from noisy telemetry
- Understanding which logs are required to detect specific behaviors
- Mapping detections to MITRE ATT&CK
- Validating detections through controlled simulations
- Investigating alerts in the Wazuh Dashboard
- Documenting detection engineering work

---

# Repository Structure

```text
wazuh-soc-home-lab/
│
├── README.md
│
├── docs/
|
├── rules/
│   ├── windows/
│   └── linux/
│   
│── screenshots/
│
└── .gitignore
```

---

# Next Steps

- Add a network telemetry source such as firewall logging or Suricata.
- Implement network scanning detection using the additional network telemetry.
- Continue improving detection coverage and MITRE ATT&CK mapping.

---
