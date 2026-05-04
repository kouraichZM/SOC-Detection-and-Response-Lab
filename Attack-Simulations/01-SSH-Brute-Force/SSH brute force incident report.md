# SSH Brute Force Attack - Incident Report

**Incident ID:** INCIDENT-2026-05-03-001  
**Date:** May 3, 2026  
**Severity:** HIGH  
**Status:** DETECTED & CONTAINED  
**Analyst:** Kouraich  
**MITRE ATT&CK:** T1110.001 (Brute Force: Password Guessing)

---

## Executive Summary

A successful SSH brute force attack was detected against the Linux web server (192.168.100.30) originating from IP address 192.168.100.54. The attacker used the THC-Hydra tool to attempt authentication with multiple user accounts and passwords. After 110 failed login attempts across 10 different usernames, the attacker successfully compromised the 'kouraich' account.

**Key Findings:**
- **110 failed SSH login attempts** detected
- **10 unique usernames** targeted (root, admin, administrator, kouraich, mysql, oracle, postgres, test, ubuntu, user)
- **1 successful compromise** of kouraich account
- **Attack duration:** ~5 minutes (19:18:56 - 19:23:15 EDT)
- **Detection time:** Real-time (< 1 minute)
- **Password cracked:** P@ssw0rd123!

---

## Attack Details

### Attack Vector
- **Protocol:** SSH (TCP Port 22)
- **Tool Used:** THC-Hydra v9.6
- **Attack Type:** Automated password brute force
- **Target Service:** OpenSSH 8.9p1 on Ubuntu 22.04.5 LTS

### Attacker Information
- **Source IP:** 192.168.100.54
- **Source System:** Kali Linux 2024.1
- **Attack Method:** Dictionary-based credential stuffing using username and password wordlists
- **Threads Used:** 4 parallel connections

### Target Information
- **Victim IP:** 192.168.100.30
- **Hostname:** kouraichwebserver
- **OS:** Ubuntu 22.04.5 LTS (GNU/Linux 5.15.0-176-generic x86_64)
- **Service:** OpenSSH Server 8.9p1
- **Compromised Account:** kouraich

---

## Attack Timeline

| Time (EDT) | Event | Details |
|------------|-------|---------|
| 19:18:56 | **Attack Initiated** | Hydra started brute force against 192.168.100.30:22 |
| 19:18:56 - 19:23:15 | **Brute Force Phase** | 110 failed authentication attempts across 10 usernames with 11 password combinations each |
| 19:23:15 | **Successful Compromise** | Password 'P@ssw0rd123!' successfully cracked for user 'kouraich' |
| 19:23:15 | **Initial Access** | Attacker established SSH session from port 42990 |
| 19:23:15 | **SIEM Detection** | Splunk correlation rule triggered HIGH severity alert |
| 19:33:35 | **Post-Compromise Activity** | Attacker logged in via SSH and accessed system information |
| 19:35:00 | **Containment** | Investigation initiated, active sessions terminated |

---

## Indicators of Compromise (IOCs)

### Network Indicators
- **Attacker IP:** 192.168.100.54
- **Target IP:** 192.168.100.30
- **Target Port:** TCP/22 (SSH)
- **Source Ports:** 42990, 48316, 56256, 35012, 35982 (multiple sessions)

### Attack Signatures
- **110 failed SSH authentication attempts** within 5-minute window
- **10 unique usernames** systematically targeted
- **11 password attempts per username** (consistent with wordlist size)
- **Rapid succession** of authentication attempts (automated tool behavior)
- **4 parallel connections** (Hydra default thread count)

### Behavioral Indicators
- Sequential username enumeration (root → admin → administrator → kouraich → mysql → oracle → postgres → test → ubuntu → user)
- Consistent password list across all usernames
- No delay between attempts (non-human behavior)
- Post-compromise SSH session immediately following successful authentication

### Compromised Credentials
- **Username:** kouraich
- **Password:** P@ssw0rd123! (weak password - violates security policy)
- **Authentication Method:** password (keyboard-interactive)

---

## Evidence

### Screenshot 1: Hydra Attack Execution

![SSH Brute Force Attack](evidence/SSH_Brute_Force_Attack.png)

**Analysis:**
- Hydra v9.6 executed with username list `/tmp/users.txt` and password list `/tmp/passwords.txt`
- Attack parameters: `-t 4` (4 threads), `-V` (verbose output)
- Total login attempts: 110 (10 users × 11 passwords)
- **Successful credential discovery:** `login: kouraich   password: P@ssw0rd123!`
- Attack completed at line [22][ssh] showing successful authentication

**Key Observations:**
- Systematic enumeration of common service accounts (root, admin, mysql, oracle, postgres)
- Weak password successfully cracked on 44th attempt
- No rate limiting or account lockout mechanisms in place

---

### Screenshot 2: Splunk Detection - Failed Authentication Attempts

![SSH Brute Force Detection](evidence/SSH_brute_Force_Attack_Detect_.png)

**Detection Query:**
```spl
index=main sourcetype="linux:auth" service="sshd" auth_result="Failed"
| stats count by src_ip, auth_user
```

**Analysis:**
- **Total failed attempts:** 110 from source IP 192.168.100.54
- **Attack pattern:** 11 attempts per username (exact match to password wordlist size)

**Targeted Usernames:**
| Username | Failed Attempts | Notes |
|----------|-----------------|-------|
| admin | 11 | Common privileged account name |
| administrator | 11 | Windows-style admin account |
| kouraich | 11 | Valid user account (successfully compromised) |
| mysql | 11 | Database service account |
| oracle | 11 | Database service account |
| postgres | 11 | Database service account |
| root | 11 | System superuser account |
| test | 11 | Common test account |
| ubuntu | 11 | Default Ubuntu installation account |
| user | 11 | Generic user account |

**Detection Effectiveness:**
- Real-time correlation detected attack within 1 minute
- Zero false positives - all 110 attempts were malicious
- Alert threshold (>5 failures) appropriately tuned

---

### Screenshot 3: Successful SSH Login After Brute Force

![SSH Successful Login](evidence/SSH_Successful_Login.png)

**Analysis:**
- **Login timestamp:** Sun May 3 19:23:15 2026 EDT
- **Authentication method:** password (successful after brute force)
- **Session established:** kouraich@kouraichwebserver
- **System information displayed:** Ubuntu 22.04.5 LTS (GNU/Linux 5.15.0-176-generic x86_64)

**Post-Compromise Access:**
- Attacker gained full shell access to web server
- System information exposed: OS version, kernel, IPv4 addresses
- User home directory accessible
- Potential for privilege escalation and lateral movement

**Security Concerns:**
- No multi-factor authentication (MFA) in place
- Password-based authentication enabled
- No SSH key-based authentication enforced
- User has potential sudo privileges (not confirmed in this session)

---

### Screenshot 4: Successful Authentication Events in Splunk

![SSH Successful Authentication](evidence/SSH_successuful_Authentication.png)

**Detection Query:**
```spl
index=main sourcetype="linux:auth" service="sshd" auth_result="Accepted"
```

**Analysis:**
- **5 successful SSH authentications** detected from 192.168.100.54
- All successful logins used 'kouraich' account
- Multiple source ports indicate separate SSH sessions

**Timeline of Successful Logins:**

| Timestamp | User | Source IP | Source Port | Event |
|-----------|------|-----------|-------------|-------|
| May 3 19:23:15 | kouraich | 192.168.100.54 | 42990 | **Initial compromise** after brute force |
| May 3 19:19:30 | kouraich | 192.168.100.54 | 48316 | Subsequent access |
| May 2 20:59:07 | kouraich | 192.168.100.54 | 56256 | Earlier legitimate login |
| May 2 20:27:45 | kouraich | 192.168.100.54 | 35012 | Earlier legitimate login |
| May 2 20:10:28 | kouraich | 192.168.100.54 | 35982 | Earlier legitimate login |

**Pattern Recognition:**
- May 2 logins (20:10, 20:27, 20:59) = Legitimate user activity
- May 3 logins (19:19, 19:23) = Attack-related activity (following brute force pattern)

**Correlation with Failed Attempts:**
```spl
index=main sourcetype="linux:auth" service="sshd" src_ip="192.168.100.54"
| stats count(eval(auth_result="Failed")) as failures,
        count(eval(auth_result="Accepted")) as successes
        by src_ip, auth_user
| where failures > 5 AND successes > 0
```

**Result:** 110 failures + 1 success = **Confirmed successful brute force attack**

---

## Detection Method

### SIEM Correlation Rule

**Alert Name:** SSH Brute Force Attack Detected  
**Rule Type:** Real-time correlation search  
**Severity:** HIGH  
**Threshold:** More than 5 failed SSH attempts from single IP within 1 hour

**Primary Detection Query:**
```spl
index=main sourcetype="linux:auth" service="sshd" auth_result="Failed" earliest=-1h
| stats count by src_ip, auth_user
| where count > 5
| sort - count
```

**Enhanced Detection Query (includes PAM failures):**
```spl
index=main sourcetype="linux:auth" service="sshd" 
(auth_result="Failed" OR auth_result="authentication failure") earliest=-1h
| stats count as total_failures,
        dc(auth_user) as unique_users_attempted,
        values(auth_user) as usernames_tried
        by src_ip
| where total_failures > 10
| sort - total_failures
```

**Alert Triggered:**
- Source IP: 192.168.100.54
- Total failures: 110
- Unique users targeted: 10
- Alert severity: HIGH
- Response time: < 1 minute from first attempt

### Detection Data Sources

| Data Source | Splunk Sourcetype | Fields Used |
|-------------|-------------------|-------------|
| Linux Authentication Logs | `linux:auth` | service, auth_result, auth_user, src_ip, src_port |
| SSH Daemon Logs | `linux:auth` | auth_method, session_action, pam_service |

### Field Extraction Configuration

**props.conf rules:**
```ini
[linux:auth]
EXTRACT-service = \s(?<service>sshd|sudo|su|CRON|login)\[?\d*\]?:
EXTRACT-auth_result = (?<auth_result>Accepted|Failed|authentication failure)
EXTRACT-auth_user = for\s+(?:invalid user\s+)?(?<auth_user>\S+?)(?:\s+from|\s+by|\s*$)
EXTRACT-src_ip = from\s+(?<src_ip>\d+\.\d+\.\d+\.\d+)
EXTRACT-src_port = port\s+(?<src_port>\d+)
```

---

## MITRE ATT&CK Mapping

### Primary Technique

| Tactic | Technique | Sub-Technique | ID |
|--------|-----------|---------------|-----|
| **Credential Access** | Brute Force | Password Guessing | **T1110.001** |

**Description:** Adversaries used THC-Hydra to systematically guess passwords for multiple user accounts via SSH authentication.

**Detection:** Authentication logs showing multiple failed login attempts from single source IP across multiple usernames.

---

### Secondary Technique

| Tactic | Technique | Sub-Technique | ID |
|--------|-----------|---------------|-----|
| **Initial Access** | Valid Accounts | Local Accounts | **T1078.003** |

**Description:** After successful password cracking, adversary authenticated using valid local account credentials.

**Detection:** Successful SSH login immediately following pattern of failed authentication attempts.

---

### Attack Chain Reconstruction

┌─────────────────────────────────────────────────────────────────┐
│ MITRE ATT&CK KILL CHAIN                                         │
└─────────────────────────────────────────────────────────────────┘

Reconnaissance (T1595.002)
└─> Port scan identified SSH service on TCP/22
Credential Access (T1110.001)
└─> Password brute force using Hydra
└─> 110 failed attempts across 10 usernames
└─> Successfully cracked: kouraich / P@ssw0rd123!
Initial Access (T1078.003)
└─> Authenticated via SSH with compromised credentials
└─> Established interactive shell session
Discovery (T1082)
└─> Viewed system information
└─> Identified OS version, kernel, network configuration
Persistence (T1098) - POTENTIAL
└─> Could create backdoor user account
└─> Could add SSH key for future access
└─> NOT OBSERVED in this incident
Lateral Movement (T1021.004) - POTENTIAL
└─> Could pivot to other systems on 192.168.100.0/24 network
└─> NOT OBSERVED in this incident

---

## Impact Assessment

### Confidentiality Impact: **HIGH**
- ✅ Unauthorized access to user account
- ✅ Exposure of system information
- ✅ Potential access to user files and data
- ✅ Exposure of network configuration
- ⚠️ Potential access to web application files (Apache, DVWA)

### Integrity Impact: **MEDIUM**
- ⚠️ Ability to modify user files
- ⚠️ Potential to modify web application code
- ⚠️ Potential to modify system configuration (if sudo access)
- ✅ No evidence of file modification observed

### Availability Impact: **LOW**
- ⚠️ Potential to terminate services
- ⚠️ Potential to consume system resources
- ✅ No denial of service observed
- ✅ System remained operational

### Overall Risk: **HIGH**
Successful compromise of user account on critical web server infrastructure with potential for lateral movement and privilege escalation.

---

## Remediation & Response Actions

### Immediate Actions Taken (Within 1 Hour)

1. ✅ **Terminated Active Sessions**
```bash
   # Kill all SSH sessions from attacker IP
   who | grep 192.168.100.54 | awk '{print $2}' | xargs -I {} pkill -t {}
```

2. ✅ **Blocked Attacker IP at Firewall**
```bash
   # Add iptables rule to block attacker
   sudo iptables -I INPUT -s 192.168.100.54 -j DROP
   sudo iptables -I OUTPUT -d 192.168.100.54 -j DROP
```

3. ✅ **Reset Compromised Password**
```bash
   # Force password change for kouraich account
   sudo passwd kouraich
   # New password: [REDACTED - meets complexity requirements]
```

4. ✅ **Reviewed Sudo Command History**
```bash
   # Check for privilege escalation attempts
   sudo grep -i "kouraich" /var/log/auth.log | grep sudo
   # Result: No sudo commands executed by attacker
```

5. ✅ **Checked File Modifications**
```bash
   # Review recent file changes
   sudo find /home/kouraich -type f -mtime -1 -ls
   # Result: No unauthorized file modifications detected
```

6. ✅ **Enabled Real-Time SSH Monitoring**
   - Splunk alert configured for immediate notification
   - Email notifications to SOC team enabled
   - Incident ticket automatically created

---

### Short-Term Remediation (Within 24 Hours)

1. ✅ **Deploy Fail2Ban**
```bash
   # Install and configure Fail2Ban
   sudo apt install fail2ban -y
   
   # Configure SSH jail
   sudo nano /etc/fail2ban/jail.local
```

   **Configuration:**
```ini
   [sshd]
   enabled = true
   port = ssh
   filter = sshd
   logpath = /var/log/auth.log
   maxretry = 3
   bantime = 3600
   findtime = 600
```

2. ✅ **Enforce Strong Password Policy**
```bash
   # Install password quality checking library
   sudo apt install libpam-pwquality -y
   
   # Edit PAM configuration
   sudo nano /etc/pam.d/common-password
```

   **Password Requirements:**
   - Minimum length: 14 characters
   - Must include: uppercase, lowercase, numbers, symbols
   - Password expiration: 90 days
   - Password history: 5 (prevent reuse)

3. ✅ **Disable Root SSH Login**
```bash
   # Edit SSH configuration
   sudo nano /etc/ssh/sshd_config
```

   **Configuration changes:**
   
   PermitRootLogin no
MaxAuthTries 3
LoginGraceTime 30

---

### Long-Term Recommendations (Within 30 Days)

1. **Deploy Multi-Factor Authentication (MFA)**
   - Implement Google Authenticator or Duo for SSH
   - Require MFA for all privileged account access
   - Test MFA implementation in lab environment first

2. **Implement Network Segmentation**
   - Isolate SSH management access to dedicated VLAN
   - Implement jump host/bastion server for SSH access
   - Restrict SSH access to specific IP ranges only

3. **Deploy Intrusion Prevention System (IPS)**
   - Configure Suricata/Snort with SSH brute force signatures
   - Implement automatic blocking at network layer
   - Integrate IPS alerts with Splunk SIEM

4. **Privileged Access Management (PAM)**
   - Deploy PAM solution for session recording
   - Implement just-in-time (JIT) privileged access
   - Centralize credential management

5. **Security Awareness Training**
   - Conduct password security training for all users
   - Implement phishing simulation exercises
   - Educate on social engineering tactics

6. **Regular Security Assessments**
   - Quarterly vulnerability scans
   - Annual penetration testing
   - Monthly password audit (check for weak passwords)
   - Review and update security policies

---

## Lessons Learned

### What Went Well ✅

1. **Real-Time Detection**
   - SIEM correlation rule detected attack within 1 minute of initiation
   - Alert threshold properly tuned (>5 failures minimized false positives)
   - Detection query captured all relevant authentication failures

2. **Comprehensive Logging**
   - All SSH authentication attempts logged to /var/log/auth.log
   - Splunk Universal Forwarder successfully forwarded logs to SIEM
   - Field extraction working correctly (src_ip, auth_user, auth_result)

3. **Rapid Investigation**
   - Complete attack timeline reconstructed from logs
   - Clear evidence chain from initial attempt to successful compromise
   - MITRE ATT&CK mapping provided context for attack techniques

4. **Effective Response**
   - Active sessions terminated immediately
   - Attacker IP blocked at firewall
   - Compromised password reset
   - No lateral movement or privilege escalation detected

---

### What Could Be Improved ❌

1. **Weak Password Policy**
   - **Issue:** Password 'P@ssw0rd123!' was easily guessable
   - **Root Cause:** No password complexity enforcement at system level
   - **Impact:** Account compromised after only 44 brute force attempts
   - **Fix:** Implement libpam-pwquality with 14-character minimum and complexity requirements

2. **No Rate Limiting**
   - **Issue:** SSH allowed unlimited rapid authentication attempts
   - **Root Cause:** No Fail2Ban or equivalent protection deployed
   - **Impact:** Attacker completed 110 attempts in 5 minutes without interruption
   - **Fix:** Deploy Fail2Ban with 3-attempt threshold and 1-hour ban time

3. **No Automated Blocking**
   - **Issue:** Attacker IP not automatically blocked after initial failures
   - **Root Cause:** No integration between SIEM alerts and firewall
   - **Impact:** Attack continued to completion despite detection
   - **Fix:** Implement SOAR playbook for automated IP blocking

4. **No Multi-Factor Authentication**
   - **Issue:** Single-factor password authentication enabled successful compromise
   - **Root Cause:** MFA not deployed on SSH service
   - **Impact:** Compromised password provided full access
   - **Fix:** Deploy Google Authenticator or Duo for SSH MFA

5. **Password-Based Authentication Enabled**
   - **Issue:** SSH accepts password authentication instead of keys only
   - **Root Cause:** Default SSH configuration allows passwords
   - **Impact:** Brute force attacks remain viable attack vector
   - **Fix:** Transition to SSH key-based authentication, disable password auth

---

### Action Items for SOC Team

| Priority | Action Item | Owner | Due Date | Status |
|----------|-------------|-------|----------|--------|
| **CRITICAL** | Deploy Fail2Ban on all SSH servers | System Admin | May 5, 2026 | ⏳ In Progress |
| **CRITICAL** | Enforce 14-character password minimum | System Admin | May 5, 2026 | ⏳ In Progress |
| **HIGH** | Implement SSH key-based authentication | System Admin | May 10, 2026 | ⏳ Planned |
| **HIGH** | Deploy MFA for privileged accounts | Security Team | May 17, 2026 | ⏳ Planned |
| **MEDIUM** | Create SOAR playbook for auto-blocking | SOC Team | May 24, 2026 | ⏳ Planned |
| **MEDIUM** | Conduct password security training | HR/Security | May 31, 2026 | ⏳ Planned |
| **LOW** | Review and update SSH hardening guide | Security Team | June 7, 2026 | ⏳ Planned |

---

## Appendix A: Tool Configuration

### Hydra Attack Command
```bash
hydra -L /tmp/users.txt -P /tmp/passwords.txt ssh://192.168.100.30 -t 4 -V
```

**Parameters:**
- `-L /tmp/users.txt` - Username wordlist
- `-P /tmp/passwords.txt` - Password wordlist
- `ssh://192.168.100.30` - Target SSH server
- `-t 4` - Use 4 parallel threads
- `-V` - Verbose output (show each attempt)

---

## Appendix B: Detection Queries

### Query 1: Real-Time Failed Attempts
```spl
index=main sourcetype="linux:auth" service="sshd" auth_result="Failed" earliest=-10m
| table _time, src_ip, auth_user, auth_method
| sort - _time
```

---

### Query 2: Brute Force Detection (Alert)
```spl
index=main sourcetype="linux:auth" service="sshd" auth_result="Failed" earliest=-1h
| stats count by src_ip, auth_user
| where count > 5
| sort - count
```
### Query 3: Timeline Visualization
```spl
index=main sourcetype="linux:auth" service="sshd" src_ip="192.168.100.54" earliest=-1h
| timechart count by auth_result
```
