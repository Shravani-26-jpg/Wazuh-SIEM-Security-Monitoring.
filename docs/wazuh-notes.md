# Wazuh SIEM & Endpoint Security Notes

A beginner-friendly technical reference guide covering Wazuh architecture, core SIEM concepts, key components, commands, and practical troubleshooting workflows.

---

## 1. What is Wazuh?

**Wazuh** is a free, open-source security platform designed for unified endpoint protection and security monitoring. It combines **Security Information and Event Management (SIEM)** with **Endpoint Detection and Response (EDR)** capabilities. 

Wazuh enables organizations and security analysts to:
- Monitor endpoints (workstations, servers, cloud instances, containers) in real time.
- Detect malware, rootkits, unauthorized system modifications, and active cyber threats.
- Analyze log data aggregated across systems to identify suspicious activities.
- Ensure regulatory compliance (e.g., PCI DSS, HIPAA, GDPR, NIST).

---

## 2. What is a SIEM?

**SIEM** stands for **Security Information and Event Management**. It is a centralized system that aggregates, correlates, and analyzes security data from multiple sources across an entire network.

Key functions of a SIEM include:
- **Log Collection & Aggregation:** Ingests event logs from endpoints, network devices, applications, and operating systems.
- **Event Correlation:** Identifies patterns across multiple events to detect complex attack chains that an individual log might not reveal.
- **Real-Time Alerting:** Generates immediate alerts when anomalous or malicious behavior matches predefined security rules.
- **Compliance Reporting & Forensics:** Stores log history for regulatory compliance audits and post-incident investigation.

---

## 3. Core Wazuh Components

Wazuh uses a modular architecture comprising four primary components:

```
[ Wazuh Agent ]  ──(Encrypted Events)──>  [ Wazuh Manager ]  ──(Parsed JSON)──>  [ Wazuh Indexer ]
(Endpoints: Linux/Win/macOS)              (Analysis & Decoders)                 (Search & Storage)
                                                                                         ▲
                                                                                         │
                                                                                [ Wazuh Dashboard ]
                                                                                (Web UI & Visualization)
```

### Wazuh Agent
The **Wazuh Agent** is lightweight client software installed on monitored endpoints (Linux, Windows, macOS, Solaris, AIX). It runs as a local background daemon (`wazuh-agentd`, `wazuh-syscheckd`, `wazuh-logcollector`, etc.) to:
- Collect system and application logs.
- Monitor file integrity and registry changes.
- Check system configuration against security benchmarks.
- Send telemetry back to the Wazuh Manager over an encrypted communication channel (port 1514/TCP).

### Wazuh Manager
The **Wazuh Manager** is the core server component responsible for intelligence and processing. It:
- Receives data sent by agents.
- Decodes and analyzes incoming logs using a rule-based detection engine.
- Matches events against threat signatures, anomaly detectors, and compliance baselines.
- Triggers active responses or alerts when threats are identified.
- Forwards parsed and indexed security events to the Wazuh Indexer.

### Wazuh Indexer
The **Wazuh Indexer** is a high-performance, distributed search and analytics engine built on OpenSearch. It:
- Stores indexed security events, alerts, and system state snapshots.
- Provides near real-time full-text search and analytical aggregation capabilities across millions of events.
- Supports clustering and data retention lifecycle policies.

### Wazuh Dashboard
The **Wazuh Dashboard** is a web-based user interface that connects to the Wazuh Indexer. It provides:
- Visual interactive dashboards and telemetry metrics for security analysts.
- Filterable views for security alerts, agent status, vulnerability assessments, and file integrity logs.
- Direct mapping of alerts to adversary tactics and techniques in the **MITRE ATT&CK** framework.
- Administration tools to configure rules, decoders, and agent groups.

---

## 4. What is File Integrity Monitoring (FIM)?

**File Integrity Monitoring (FIM)** is an internal security mechanism (implemented by the `wazuh-syscheckd` daemon in Wazuh) that continuously inspects files and directories for unauthorized changes.

FIM tracks:
- **File creation:** Detects when new files or suspicious executables are created.
- **File modification:** Detects alterations in file content by calculating and comparing cryptographic hashes (MD5, SHA-1, SHA-256).
- **Attribute and permission changes:** Detects modifications to file ownership, read/write/execute permissions, timestamps, or system attributes.
- **File deletion:** Detects removal of critical configuration files or binary payloads.

FIM is configured in the agent or manager configuration file (`ossec.conf`) within the `<syscheck>` block.

---

## 5. What is MITRE ATT&CK?

**MITRE ATT&CK** (Adversarial Tactics, Techniques, and Common Knowledge) is a globally recognized, curated knowledge base of adversary behaviors based on real-world cyber observations.

In Wazuh:
- Incoming alerts are automatically mapped to corresponding MITRE ATT&CK tactics (e.g., *Initial Access*, *Persistence*, *Privilege Escalation*, *Defense Evasion*, *Exfiltration*) and specific technique IDs (e.g., `T1078`, `T1059`).
- This allows Security Operations Center (SOC) analysts to quickly categorize threat events, understand attacker intent, and determine remediation actions according to industry-standard frameworks.

---

## 6. Common Wazuh Management Commands

### Wazuh Manager (Server)

```bash
# Check status of the Wazuh Manager service
sudo systemctl status wazuh-manager

# Start / Stop / Restart Wazuh Manager
sudo systemctl start wazuh-manager
sudo systemctl stop wazuh-manager
sudo systemctl restart wazuh-manager

# View Wazuh Manager logs in real time
sudo tail -f /var/ossec/logs/ossec.log

# List connected and registered agents via Wazuh CLI
sudo /var/ossec/bin/agent_control -l

# Check active processes and status of Wazuh manager daemons
sudo /var/ossec/bin/wazuh-control status
```

### Wazuh Agent (Endpoint)

```bash
# Check status of the Wazuh Agent service
sudo systemctl status wazuh-agent

# Start / Stop / Restart Wazuh Agent
sudo systemctl start wazuh-agent
sudo systemctl stop wazuh-agent
sudo systemctl restart wazuh-agent

# Check systemd journal logs for wazuh-agent
sudo journalctl -u wazuh-agent --no-pager -n 30

# Inspect agent runtime logs
sudo tail -30 /var/ossec/logs/ossec.log

# Validate agent configuration file syntax
sudo /var/ossec/bin/wazuh-agent-check-conf /var/ossec/etc/ossec.conf
```

---

## 7. Troubleshooting Notes

### 1. `Invalid server address found: 'MANAGER_IP'`
- **Symptom:** Wazuh agent fails to start; `systemctl status wazuh-agent` shows status failure, and `/var/ossec/logs/ossec.log` displays:
  ```text
  wazuh-agentd: ERROR: (4112): Invalid server address found: 'MANAGER_IP'
  wazuh-agentd: ERROR: (1215): No client configured. Exiting.
  ```
- **Cause:** The agent package was installed without supplying the manager's IP address, leaving the default placeholder `'MANAGER_IP'` in the configuration.
- **Resolution:**
  1. Open `/var/ossec/etc/ossec.conf` with root privileges.
  2. Locate `<client><server><address>MANAGER_IP</address></server></client>`.
  3. Replace `MANAGER_IP` with the actual IP address or reachable FQDN of the Wazuh Manager (e.g., `172.27.18.120`).
  4. Save the file and restart the service:
     ```bash
     sudo systemctl restart wazuh-agent
     ```

### 2. Permission Denied During Repository / Key Setup
- **Symptom:** Running repository enrollment commands produces:
  ```text
  gpg: failed to create temporary file '/usr/share/keyrings/...': Permission denied
  tee: /etc/apt/sources.list.d/wazuh.list: Permission denied
  ```
- **Cause:** Writing to protected system directories (`/usr/share/keyrings` and `/etc/apt/sources.list.d`) requires root privileges.
- **Resolution:** Prefix administrative commands with `sudo` or switch to the root shell:
  ```bash
  curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | sudo gpg --no-default-keyring --keyring gnupg-ring:/usr/share/keyrings/wazuh.gpg --import
  sudo chmod 644 /usr/share/keyrings/wazuh.gpg
  echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main" | sudo tee /etc/apt/sources.list.d/wazuh.list
  sudo apt-get update
  ```

### 3. Package Format Mismatch on Linux Distributions
- **Symptom:** Attempting to install an `.rpm` package on a Debian/Ubuntu-derived system (like Kali Linux) results in failed dependencies (`/bin/sh`, `groupadd`, `useradd` needed).
- **Resolution:** Ensure the correct package format is chosen:
  - Use `.deb` packages for Debian, Ubuntu, Kali Linux, and Linux Mint (`apt` / `dpkg`).
  - Use `.rpm` packages for RHEL, CentOS, Fedora, Rocky Linux, and AlmaLinux (`yum` / `dnf` / `rpm`).
