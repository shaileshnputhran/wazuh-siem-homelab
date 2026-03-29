# Investigation Report — File Integrity Monitoring (FIM)
**Analyst:** Shailesh Nagesh Puthran  
**Date:** March 29, 2026  
**Agent:** Mac.lan (ID: 001) — macOS 26.3  
**Module:** Wazuh File Integrity Monitoring  
**Severity:** 🟠 High  

---

## 1. Scenario

The Wazuh FIM module detected multiple file modification events on critical system binaries and directories on the macOS endpoint. FIM monitors files for any changes to content, permissions, ownership or attributes — alerting the SOC when monitored files are modified, added or deleted. This investigation was triggered by a spike in FIM events detected in the dashboard.

---

## 2. Data Sources Used

| Source | Details |
|---|---|
| SIEM Platform | Wazuh 4.10.3 |
| Module | File Integrity Monitoring (syscheck) |
| Agent | Mac.lan (001) — macOS 26.3 |
| Rule Group | syscheck, syscheck_file, syscheck_entry_mo... |
| Rule ID | 550 — Integrity checksum changed |
| Detection period | Last 24 hours |

---

## 3. Investigation Timeline

**Step 1 — Alert triage**  
Navigated to Wazuh Dashboard → File Integrity Monitoring → Mac.lan agent. FIM dashboard showed a large spike in modification events at approximately 05:18 UTC.

**Step 2 — Identified most active user**  
Reviewed the "Most active users" panel:
- `root` — 99.45% of all file modification events
- `_uucp` — 0.55% of events

The overwhelming dominance of `root` making file changes is a significant indicator — legitimate system updates typically show a mix of users and processes.

**Step 3 — Identified action types**  
All detected events were of type `modified` (100%). No file additions or deletions were recorded, suggesting targeted modification of existing files rather than new file drops.

**Step 4 — Reviewed modified file paths**  
Examined the specific files that were modified:

| Time (UTC) | File Path | Action | Rule ID |
|---|---|---|---|
| 05:18:57.236 | /usr/sbin/htcacheclean | modified | 550 |
| 05:18:57.234 | /usr/sbin/apachectl | modified | 550 |
| 05:18:57.230 | /usr/sbin/WirelessRadioManagerd | modified | 550 |
| 05:18:57.219 | /usr/sbin/cvadmin | modified | 550 |
| 05:18:57.216 | /usr/sbin/auditreduce | modified | 550 |
| 05:18:xx | /bin/bash | modified | 550 |
| 05:18:xx | /bin/cat | modified | 550 |
| 05:18:xx | /bin/chmod | modified | 550 |
| 05:18:xx | /bin/cp | modified | 550 |

**Step 5 — Assessed criticality of modified files**  
The modified files fall into two high-risk categories:

*Critical system binaries (/bin/)* — bash, cat, chmod, cp are core Unix utilities. Modification of these binaries is a classic indicator of:
- Rootkit installation (replacing system binaries with malicious versions)
- Trojanised binaries (adding backdoor functionality)
- Attacker persistence mechanisms

*System daemons (/usr/sbin/)* — apachectl, WirelessRadioManagerd, auditreduce are system-level services. Modification of `auditreduce` is particularly suspicious as it handles audit log processing — a common target for attackers attempting to cover tracks.

**Step 6 — Contextualised with macOS update behaviour**  
After analysis, these modifications were consistent with a macOS system update running at 05:18 UTC — Apple system updates legitimately modify these binaries under the root user. However, in a real SOC environment, this would need to be correlated with a confirmed change management ticket before being closed as a false positive.

---

## 4. Indicators of Compromise (IOCs)

| Type | Value | Risk Level |
|---|---|---|
| Modified binary | /bin/bash | 🔴 Critical — shell interpreter |
| Modified binary | /bin/cat | 🟠 High — core utility |
| Modified binary | /bin/chmod | 🟠 High — permission tool |
| Modified binary | /bin/cp | 🟡 Medium — copy utility |
| Modified daemon | /usr/sbin/apachectl | 🟠 High — web server control |
| Modified daemon | /usr/sbin/auditreduce | 🔴 Critical — audit log tool |
| Modified daemon | /usr/sbin/WirelessRadioManagerd | 🟡 Medium — wireless manager |

---

## 5. MITRE ATT&CK Mapping

| Technique ID | Technique Name | Relevance |
|---|---|---|
| T1565.001 | Stored Data Manipulation | Modification of binary files on disk |
| T1070.002 | Clear Linux or Mac System Logs | auditreduce modification could indicate log tampering |
| T1543.001 | Launch Agent | Modification of system daemons for persistence |
| T1059.004 | Unix Shell | Modification of /bin/bash — shell interpreter |
| T1036.005 | Match Legitimate Name or Location | Replacing legit binaries with malicious ones |

---

## 6. Risk Assessment

| Factor | Assessment |
|---|---|
| **Exploitability** | High — if malicious, modified binaries execute with system privileges |
| **Impact** | Critical — /bin/bash modification could give attacker persistent shell access |
| **Likelihood** | Low — likely macOS system update (contextual analysis) |
| **Overall Risk** | 🟠 MEDIUM (pending change management verification) |

---

## 7. Verdict

**LIKELY FALSE POSITIVE — Verify against change management**

The file modification events are consistent with a legitimate macOS system update running at 05:18 UTC. However, this cannot be confirmed without cross-referencing against:
1. A confirmed macOS update event in system logs
2. A change management ticket authorising the update
3. File hash verification against Apple's known-good binary hashes

In a real SOC, this would be closed as a false positive only after those three checks pass. If any check fails, it escalates to a True Positive incident.

---

## 8. Recommended Actions

1. **Verify:** Cross-reference with macOS Software Update logs: `log show --predicate 'subsystem == "com.apple.SoftwareUpdate"' --last 24h`
2. **Verify:** Check file hashes against Apple's baseline for the installed macOS version
3. **Document:** If confirmed as system update, document in ticket as False Positive with justification
4. **Tune:** Create a FIM exclusion rule for known Apple update processes to reduce alert noise
5. **Monitor:** Continue monitoring — if modifications recur outside of update windows, re-escalate

---

## 9. Lessons Learned

- FIM generates high alert volumes during system updates — SOC teams must correlate with change management
- Modification of audit tools like `auditreduce` should always be treated as high priority
- `/bin/bash` modifications are never to be dismissed without thorough verification
- Alert tuning is essential in production SIEM environments to separate legitimate changes from malicious ones

---

*Report generated from Wazuh 4.10.3 File Integrity Monitoring module — Mac.lan agent (001)*
