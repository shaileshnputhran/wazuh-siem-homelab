# 🛡️ Wazuh SIEM Home Lab — macOS (M2 Apple Silicon)

> **Built by:** Shailesh Nagesh Puthran | SOC Analyst (L1) | Dubai, UAE  
> **Status:** ✅ Fully operational  
> **Platform:** macOS (Apple Silicon M2) + Docker Desktop  
> **Wazuh Version:** 4.10.3  

---

## 📋 Project Overview

This project documents the build and operation of a fully functional **enterprise-grade SIEM home lab** running Wazuh 4.10.3 on an Apple Silicon M2 Mac using Docker Desktop. The lab simulates a real SOC monitoring environment — complete with a monitored endpoint, live alert generation, vulnerability detection, file integrity monitoring, and MITRE ATT&CK mapped detections.

This lab was built to develop and demonstrate hands-on SOC analyst skills including:
- SIEM deployment and configuration
- Endpoint agent enrollment and monitoring
- Alert triage and investigation
- Vulnerability management
- File Integrity Monitoring (FIM)
- Security Configuration Assessment (SCA)
- Custom detection rule writing
- MITRE ATT&CK technique mapping

---

## 🏗️ Lab Architecture

```
┌─────────────────────────────────────────────────────┐
│                   M2 MacBook                        │
│                                                     │
│  ┌─────────────────────────────────────────────┐   │
│  │           Docker Desktop                    │   │
│  │                                             │   │
│  │  ┌─────────────┐   ┌─────────────────────┐ │   │
│  │  │   wazuh.    │   │    wazuh.manager    │ │   │
│  │  │   indexer   │◄──│  (Detection Engine) │ │   │
│  │  │ (OpenSearch)│   └─────────────────────┘ │   │
│  │  └──────┬──────┘            ▲               │   │
│  │         │                   │               │   │
│  │  ┌──────▼──────┐            │               │   │
│  │  │   wazuh.    │            │               │   │
│  │  │  dashboard  │            │               │   │
│  │  │ (Web UI 443)│            │               │   │
│  │  └─────────────┘            │               │   │
│  └─────────────────────────────┼───────────────┘   │
│                                │                    │
│  ┌─────────────────────────────▼───────────────┐   │
│  │         Wazuh Agent (Mac endpoint)          │   │
│  │   /Library/Ossec/ — monitoring macOS 26.3   │   │
│  └─────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────┘

Dashboard URL: https://localhost:443
Agent ID:      001 (Mac.lan) — Status: Active
```

---

## 🛠️ Tools & Technologies

| Component | Technology | Purpose |
|---|---|---|
| SIEM Platform | Wazuh 4.10.3 | Central security monitoring |
| Search & Storage | OpenSearch (Wazuh Indexer) | Alert indexing and search |
| Visualisation | Wazuh Dashboard | Web UI, dashboards, reports |
| Containerisation | Docker Desktop (Apple Silicon) | Running Wazuh stack |
| Endpoint Agent | Wazuh Agent ARM64 | Monitoring Mac endpoint |
| Framework | MITRE ATT&CK | Technique mapping |
| Compliance | PCI DSS, NIST 800-53 | Compliance monitoring |

---

## ⚙️ Setup & Installation

### Prerequisites
- macOS (Apple Silicon M2/M3/M4)
- Docker Desktop for Mac (Apple Silicon) — `docker.com/products/docker-desktop`
- At least 8 GB RAM available for Docker
- Git installed

### Step 1 — Configure Docker memory
Open Docker Desktop → Settings → Resources → set Memory to **6 GB minimum**. Click Apply & Restart.

### Step 2 — Set vm.max_map_count (required for OpenSearch)
```bash
docker run --rm --privileged --pid=host alpine nsenter -t 1 -m -u -n -i sysctl -w vm.max_map_count=262144
```

### Step 3 — Clone Wazuh Docker repository
```bash
cd ~/Downloads
git clone https://github.com/wazuh/wazuh-docker.git -b v4.10.3
cd wazuh-docker/single-node
```

### Step 4 — Generate SSL certificates
```bash
docker compose -f generate-indexer-certs.yml run --rm generator
```

### Step 5 — Start Wazuh
```bash
docker compose up -d
```

### Step 6 — Verify all containers are running
```bash
docker compose ps
```

Expected output:
```
single-node-wazuh.dashboard-1   Up    0.0.0.0:443->5601/tcp
single-node-wazuh.indexer-1     Up    0.0.0.0:9200->9200/tcp
single-node-wazuh.manager-1     Up    0.0.0.0:1514-1515->1514-1515/tcp
```

### Step 7 — Access the Dashboard
Open browser → `https://localhost:443`  
Accept the certificate warning → Login: `admin` / `SecretPassword`

### Step 8 — Install Wazuh Agent on Mac (ARM64)
```bash
curl -so wazuh-agent.pkg https://packages.wazuh.com/4.x/macos/wazuh-agent-4.10.3-1.arm64.pkg
sudo WAZUH_MANAGER='127.0.0.1' installer -pkg wazuh-agent.pkg -target /
```

Fix the manager IP in config:
```bash
sudo nano /Library/Ossec/etc/ossec.conf
# Change <address>MANAGER_IP</address> to <address>127.0.0.1</address>
```

Remove unsupported syscollector tags:
```bash
sudo sed -i '' '/<users>/d; /<groups>/d' /Library/Ossec/etc/ossec.conf
```

Start the agent:
```bash
sudo /Library/Ossec/bin/wazuh-control start
```

### To start Wazuh after a Mac reboot
```bash
cd ~/Downloads/wazuh-docker/single-node
docker compose up -d
sudo /Library/Ossec/bin/wazuh-control start
```

---

## 🔬 Lab Exercises & Findings

### Exercise 1 — Vulnerability Detection Scan
**Objective:** Identify CVEs on the monitored Mac endpoint  
**Module:** Wazuh → Vulnerability Detection  
**Method:** Automated vulnerability scan run against Mac.lan agent  

**Findings:**

| Severity | Count |
|---|---|
| 🔴 Critical | 2 |
| 🟠 High | 22 |
| 🟡 Medium | 48 |
| 🟢 Low | 5 |

**Top affected packages:** macOS (58 packages), pip (10), setuptools (6), Safari (5)

**MITRE ATT&CK Mapping:** T1190 — Exploit Public-Facing Application

**Investigation notes:** Two critical CVEs identified on the macOS endpoint. In a real SOC environment these would be escalated immediately to the vulnerability management team for patching prioritisation. The pip and setuptools vulnerabilities indicate outdated Python packages that should be updated.

---

### Exercise 2 — File Integrity Monitoring (FIM)
**Objective:** Detect unauthorised file modifications on the endpoint  
**Module:** Wazuh → File Integrity Monitoring  
**Method:** Wazuh FIM module continuously monitors critical system directories  

**Findings:**
- Most active user making changes: `root` (99.45%)
- Action type: `modified` (100% of events)
- Files monitored with changes detected:

| File Path | Action | Rule ID |
|---|---|---|
| /usr/sbin/htcacheclean | modified | 550 |
| /usr/sbin/apachectl | modified | 550 |
| /usr/sbin/WirelessRadioManagerd | modified | 550 |
| /bin/bash | modified | 550 |
| /bin/cat | modified | 550 |
| /bin/chmod | modified | 550 |
| /bin/cp | modified | 550 |

**MITRE ATT&CK Mapping:** T1565 — Data Manipulation, T1070 — Indicator Removal

**Investigation notes:** System binary modifications detected. In a real incident, modifications to `/bin/bash` and `/bin/cat` would be high-priority indicators of potential trojanised binaries or rootkit activity. These events would be immediately escalated as a potential True Positive.

---

### Exercise 3 — Security Configuration Assessment (SCA)
**Objective:** Identify security misconfigurations on the endpoint  
**Module:** Wazuh → Configuration Assessment  
**Policy:** System audit for Unix based systems  

**Findings:**

| Result | Count |
|---|---|
| ✅ Passed | 0 |
| ❌ Failed | 10 |
| ⚪ Not applicable | 13 |
| Score | 0% |

**SSH Hardening failures detected:**

| Check ID | Finding | Risk |
|---|---|---|
| 3000 | SSH Port should not be 22 | Medium |
| 3001 | SSH Protocol should be set to 2 | High |
| 3002 | Root account should not be allowed via SSH | High |
| 3003 | No Public Key authentication configured | Medium |
| 3004 | Password Authentication should be disabled | High |
| 3005 | Empty passwords should not be permitted | Critical |
| 3006 | Rhost or shost should not be allowed | High |
| 3007 | Grace Time should be one minute | Low |
| 3008 | Wrong Maximum number of authentication attempts | Medium |
| 3009 | SSH HostbasedAuthentication not disabled | High |

**MITRE ATT&CK Mapping:** T1110 — Brute Force, T1021.004 — SSH

**Investigation notes:** The SSH configuration on this endpoint fails all 10 hardening checks. In a real SOC environment, this would generate a remediation ticket. Key risks: password authentication enabled with no brute force protection, root SSH login permitted, and empty passwords allowed — all of which would make this endpoint highly vulnerable to credential-based attacks.

---

### Exercise 4 — MITRE ATT&CK Threat Detection
**Objective:** Identify adversary techniques detected by Wazuh  
**Module:** Wazuh → Threat Hunting → MITRE ATT&CK panel  

**Top tactics detected:**

| Tactic | Alert Count | Significance |
|---|---|---|
| Impact | 1,266 | System modifications affecting integrity |
| Defense Evasion | 6 | Attempts to avoid detection |
| Privilege Escalation | 3 | Attempts to gain higher privileges |
| Initial Access | 2 | Entry point techniques |
| Persistence | 2 | Mechanisms to maintain access |

**Total alerts in 24 hours:** 1,373  
**High severity (Level 12+):** 0  
**Medium severity alerts:** 1,397  

---

## 📁 Repository Structure

```
wazuh-siem-homelab/
├── README.md                          ← This file
├── setup/
│   ├── docker-compose-notes.md        ← Setup notes and troubleshooting
│   └── agent-installation-macos.md   ← Agent setup steps for macOS ARM64
├── detection-rules/
│   └── local_rules.xml                ← Custom Wazuh detection rules
├── investigations/
│   ├── vulnerability-scan-analysis.md
│   ├── file-integrity-monitoring.md
│   ├── ssh-misconfiguration-analysis.md
│   └── mitre-attck-detections.md
└── screenshots/
    ├── 01-overview-dashboard.png
    ├── 02-agents-active.png
    ├── 03-agent-detail-mitre.png
    ├── 04-threat-hunting.png
    ├── 05-file-integrity-monitoring.png
    └── 06-configuration-assessment.png
```

---

## 🧠 Custom Detection Rule

Written to detect suspicious `whoami` command execution — mapped to MITRE T1033 (System Owner/User Discovery):

```xml
<group name="custom_rules,">
  <rule id="100001" level="5">
    <if_group>sudo</if_group>
    <match>whoami</match>
    <description>Suspicious whoami command via sudo detected — possible reconnaissance</description>
    <mitre>
      <id>T1033</id>
    </mitre>
  </rule>
</group>
```

---

## 📊 Lab Dashboard Screenshots

| Screenshot | Description |
|---|---|
| `screenshots/01-overview-dashboard.png` | Wazuh overview — 1 active agent, 1,559 total alerts |
| `screenshots/02-agents-active.png` | Mac.lan agent Active — macOS 26.3, Wazuh v4.10.3 |
| `screenshots/03-agent-detail-mitre.png` | MITRE ATT&CK detections + CVE findings + FIM events |
| `screenshots/04-threat-hunting.png` | 1,373 events — ossec, syscheck, sudo, authentication groups |
| `screenshots/05-file-integrity-monitoring.png` | FIM — /bin/bash, /bin/cat, /bin/chmod modifications detected |
| `screenshots/06-configuration-assessment.png` | SCA — 10 SSH hardening failures identified |

---

## 🎯 Key Skills Demonstrated

- Deployed a multi-component SIEM stack (Indexer + Manager + Dashboard) using Docker on macOS Apple Silicon
- Enrolled and configured a macOS endpoint as a monitored Wazuh agent
- Identified and documented **2 Critical CVEs** and **22 High CVEs** through vulnerability scanning
- Detected and analysed **file integrity violations** on critical system binaries
- Identified **10 SSH hardening failures** through Security Configuration Assessment
- Mapped detected events to **5 MITRE ATT&CK tactics** including Privilege Escalation and Defense Evasion
- Wrote a **custom Wazuh detection rule** mapped to MITRE T1033
- Produced professional investigation reports for each finding

---

## 🔗 Connect

| | |
|---|---|
| 📍 Location | Dubai, UAE |
| 💼 LinkedIn | [linkedin.com/in/YOUR-LINKEDIN](https://linkedin.com/in/YOUR-LINKEDIN-HERE) |
| 🏆 TryHackMe | [tryhackme.com/p/YOUR-USERNAME](https://tryhackme.com/p/YOUR-USERNAME-HERE) |
| 📧 Email | your.email@gmail.com |

---

> *This home lab was built entirely on an Apple Silicon M2 Mac without any cloud resources. All findings are from real detections on a live monitored endpoint — not simulated data.*
