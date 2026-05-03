# SOC Home Lab – Detection, Monitoring & Incident Response

This repository documents a self-built SOC-focused home lab designed to simulate a realistic enterprise environment and practice security monitoring, detection, investigation, and incident response.

The goal of this lab is not exploitation for its own sake, but understanding how attacks appear in logs, how they are detected in a SIEM, and how a SOC analyst investigates and responds.

---

## 🧱 Lab Architecture

**Operating Systems & Roles**

* **Windows Server 2019/2022**
   * Active Directory Domain Services
   * DNS
   * DHCP
* **Windows 10 Pro**
   * Domain-joined endpoint
   * User activity & endpoint telemetry
* **Ubuntu Server**
   * Apache Web Server
   * Vulnerable Web Application (DVWA)
* **Splunk SIEM**
   * Centralized log collection & alerting
* **Kali Linux**
   * Controlled attack simulation
* **Sysmon**
   * Enhanced Windows telemetry

All systems are deployed in a controlled internal network to replicate an enterprise SOC environment.

---

## 🎯 Objectives of the Lab

* Build hands-on experience with SOC workflows
* Understand how attacks look from a detection and logging perspective
* Practice troubleshooting AD, DNS, DHCP, and networking
* Generate and analyze realistic security telemetry
* Investigate alerts and produce SOC-style incident reports
* Map detections to MITRE ATT&CK

---

## 🔍 Current Capabilities

* Centralized logging from:
   * Windows endpoints
   * Domain Controller
   * Linux server
   * Apache web services
* Endpoint visibility using Sysmon
* Detection of:
   * Suspicious PowerShell execution
   * Authentication failures
   * Network scanning activity
   * Web attacks (SQLi, command injection, XSS via DVWA)
* Correlation of attacker activity across multiple hosts
* Incident investigation and documentation

---

## 🧪 Attack Scenarios Implemented

* PowerShell-based initial access simulation
* Local and domain reconnaissance
* Internal network scanning
* Authentication abuse (controlled)
* Web application attacks against DVWA
* Log correlation between web, host, and SIEM layers

Each scenario is designed to be safe, non-destructive, and focused on detection and analysis.

---

## 📊 Incident Response & Analysis

For each scenario, the following are documented:

* Attack summary
* Timeline of events
* Affected hosts and users
* Relevant logs and alerts
* MITRE ATT&CK techniques
* Severity assessment
* Recommended remediation actions

---

## 🛠️ Troubleshooting & Lessons Learned

Key lessons from building and operating this lab:

* AD, DNS, and DHCP misconfigurations can break an entire environment
* DNS resolution is foundational to domain functionality
* SIEMs require intentional log engineering
* Networking and routing issues are a core SOC troubleshooting skill
* Visibility matters more than exploitation complexity

---

## 🚀 Planned Enhancements

* Firewall integration (pfSense / OPNsense)
* IDS/IPS alerts (Suricata)
* Custom Splunk detection rules
* Threat hunting queries
* Advanced Active Directory attack detection
* Detection tuning and false-positive reduction

---

## ⚠️ Disclaimer

This lab is conducted entirely in an isolated environment for defensive security learning purposes only. No systems or networks outside this lab were targeted.
