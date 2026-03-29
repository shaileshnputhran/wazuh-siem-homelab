# Investigation Report — SSH Misconfiguration Analysis (SCA)
**Analyst:** Shailesh Nagesh Puthran  
**Date:** March 29, 2026  
**Agent:** Mac.lan (ID: 001) — macOS 26.3  
**Module:** Wazuh Security Configuration Assessment (SCA)  
**Severity:** 🔴 Critical  

---

## 1. Scenario

The Wazuh Security Configuration Assessment module ran an automated audit against the macOS endpoint using the "System audit for Unix based systems" policy. The scan completed with a score of 0% — all applicable checks failed. This investigation documents the findings, assesses the risk, and provides remediation guidance for each failed check.

---

## 2. Data Sources Used

| Source | Details |
|---|---|
| SIEM Platform | Wazuh 4.10.3 |
| Module | Security Configuration Assessment (SCA) |
| Agent | Mac.lan (001) — macOS 26.3 |
| Policy | System audit for Unix based systems (unix_audit) |
| Scan completed | March 29, 2026 @ 05:18:20 UTC |
| Score | 0% (0 passed, 10 failed, 13 not applicable) |

---

## 3. Investigation Timeline

**Step 1 — Alert triage**  
Navigated to Wazuh Dashboard → Configuration Assessment → Mac.lan. The SCA summary immediately showed a 0% compliance score — a critical finding indicating the endpoint has no SSH hardening in place.

**Step 2 — Reviewed policy results**  
Expanded the "System audit for Unix based systems" policy to view all 23 checks. 10 checks failed, 13 were not applicable to this macOS version.

**Step 3 — Categorised failures**  
All 10 failures were SSH hardening related — targeting the `/etc/ssh/sshd_config` file. This means the SSH service is running with default, insecure settings, exposing the endpoint to multiple attack vectors.

**Step 4 — Assessed each failure individually**  
Reviewed each failed check against its security impact and attack scenario.

**Step 5 — Prioritised by risk**  
Ranked failures by exploitability and potential impact to determine remediation order.

---

## 4. Detailed Findings

### Check 3000 — SSH Port should not be 22
| Field | Value |
|---|---|
| **Target file** | /etc/ssh/sshd_config |
| **Result** | ❌ Failed |
| **Risk** | Medium |
| **Why it matters** | Port 22 is the default SSH port and is constantly scanned by automated bots and threat actors. Running SSH on a non-standard port significantly reduces automated attack noise. |
| **Fix** | Change `Port 22` to a non-standard port (e.g. 2222) in sshd_config |

---

### Check 3001 — SSH Protocol should be set to 2
| Field | Value |
|---|---|
| **Target file** | /etc/ssh/sshd_config |
| **Result** | ❌ Failed |
| **Risk** | High |
| **Why it matters** | SSH Protocol 1 has known cryptographic vulnerabilities including man-in-the-middle attacks. Protocol 2 uses stronger encryption and authentication. |
| **Fix** | Add `Protocol 2` to sshd_config |

---

### Check 3002 — Root account should not be allowed via SSH
| Field | Value |
|---|---|
| **Target file** | /etc/ssh/sshd_config |
| **Result** | ❌ Failed |
| **Risk** | High |
| **Why it matters** | Allowing root login via SSH means a successful brute force attack immediately grants full system control with no privilege escalation step required. |
| **Fix** | Set `PermitRootLogin no` in sshd_config |

---

### Check 3003 — No Public Key authentication configured
| Field | Value |
|---|---|
| **Target file** | /etc/ssh/sshd_config |
| **Result** | ❌ Failed |
| **Risk** | Medium |
| **Why it matters** | Public key authentication is significantly stronger than password-based authentication. Without it, the system relies solely on passwords which are vulnerable to brute force and credential stuffing. |
| **Fix** | Set `PubkeyAuthentication yes` and configure SSH keys |

---

### Check 3004 — Password Authentication should be disabled
| Field | Value |
|---|---|
| **Target file** | /etc/ssh/sshd_config |
| **Result** | ❌ Failed |
| **Risk** | 🔴 Critical |
| **Why it matters** | Password authentication is the primary target of brute force and credential stuffing attacks. With password auth enabled and no account lockout, an attacker can attempt unlimited password guesses. |
| **Fix** | Set `PasswordAuthentication no` — only allow key-based auth |

---

### Check 3005 — Empty passwords should not be permitted
| Field | Value |
|---|---|
| **Target file** | /etc/ssh/sshd_config |
| **Result** | ❌ Failed |
| **Risk** | 🔴 Critical |
| **Why it matters** | Allowing empty passwords means any account without a password set can be accessed via SSH with no credentials whatsoever — a trivially exploitable misconfiguration. |
| **Fix** | Set `PermitEmptyPasswords no` in sshd_config |

---

### Check 3006 — Rhost or shost should not be allowed
| Field | Value |
|---|---|
| **Target file** | /etc/ssh/sshd_config |
| **Result** | ❌ Failed |
| **Risk** | High |
| **Why it matters** | Rhosts-based authentication trusts remote hosts by IP address — a legacy mechanism vulnerable to IP spoofing attacks. |
| **Fix** | Set `IgnoreRhosts yes` and `RhostsRSAAuthentication no` |

---

### Check 3007 — Grace Time should be one minute
| Field | Value |
|---|---|
| **Target file** | /etc/ssh/sshd_config |
| **Result** | ❌ Failed |
| **Risk** | Low |
| **Why it matters** | LoginGraceTime controls how long a connection is held open during authentication. A long grace time allows more time for brute force attempts per connection. |
| **Fix** | Set `LoginGraceTime 60` in sshd_config |

---

### Check 3008 — Wrong Maximum number of authentication attempts
| Field | Value |
|---|---|
| **Target file** | /etc/ssh/sshd_config |
| **Result** | ❌ Failed |
| **Risk** | Medium |
| **Why it matters** | Without limiting authentication attempts, brute force tools can make unlimited guesses per connection. Setting MaxAuthTries to 3-4 significantly slows brute force attacks. |
| **Fix** | Set `MaxAuthTries 3` in sshd_config |

---

### Check 3009 — SSH HostbasedAuthentication not disabled
| Field | Value |
|---|---|
| **Target file** | /etc/ssh/sshd_config |
| **Result** | ❌ Failed |
| **Risk** | High |
| **Why it matters** | HostbasedAuthentication trusts other hosts to authenticate users — vulnerable to host spoofing and lateral movement attacks in a network. |
| **Fix** | Set `HostbasedAuthentication no` in sshd_config |

---

## 5. MITRE ATT&CK Mapping

| Technique ID | Technique Name | Related Check |
|---|---|---|
| T1110.001 | Brute Force: Password Guessing | Checks 3004, 3005, 3008 |
| T1110.003 | Brute Force: Password Spraying | Check 3004 |
| T1021.004 | Remote Services: SSH | All checks |
| T1078 | Valid Accounts | Check 3002 — root login |
| T1563.001 | Remote Service Session Hijacking: SSH | Check 3001 — Protocol 1 |
| T1599 | Network Boundary Bridging | Checks 3006, 3009 — rhost auth |

---

## 6. Risk Assessment

| Factor | Assessment |
|---|---|
| **Exploitability** | 🔴 Critical — multiple trivially exploitable misconfigs |
| **Impact** | 🔴 Critical — successful attack grants full system access |
| **Likelihood** | Medium — endpoint not directly internet-exposed in lab |
| **Overall Risk** | 🔴 CRITICAL |

---

## 7. Verdict

**TRUE POSITIVE — Immediate remediation required**

The endpoint has 10 SSH hardening failures with 0% compliance score. The most critical findings are:
- Password authentication enabled (Check 3004) — allows brute force
- Empty passwords permitted (Check 3005) — allows passwordless access
- Root login allowed (Check 3002) — no privilege escalation needed post-compromise

In a production environment, this endpoint would be immediately flagged for emergency remediation and potentially isolated from the network pending patching.

---

## 8. Recommended Remediation

Apply all fixes by editing the SSH config file:
```bash
sudo nano /etc/ssh/sshd_config
```

Add or update these settings:
```
Port 2222
Protocol 2
PermitRootLogin no
PubkeyAuthentication yes
PasswordAuthentication no
PermitEmptyPasswords no
IgnoreRhosts yes
LoginGraceTime 60
MaxAuthTries 3
HostbasedAuthentication no
```

Restart SSH after changes:
```bash
sudo launchctl stop com.openssh.sshd
sudo launchctl start com.openssh.sshd
```

Re-run the SCA scan to verify remediation:
- Wazuh Dashboard → Configuration Assessment → Refresh

**Expected result after remediation:** Score should rise from 0% to 70%+ with all 10 checks passing.

---

## 9. Lessons Learned

- Default SSH configurations on macOS are highly insecure and should never be used in production
- A 0% SCA score on first scan is normal for unconfigured systems — the value is in identifying and fixing the gaps
- Password authentication should always be disabled in favour of key-based authentication
- Empty password checks should be part of every endpoint onboarding checklist
- Wazuh SCA provides automated CIS/hardening benchmark compliance — eliminating the need for manual configuration reviews

---

*Report generated from Wazuh 4.10.3 Security Configuration Assessment module — Mac.lan agent (001)*
