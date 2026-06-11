<div align="center">

# 🛡️ Adversary Emulation & Detection Lab

**Course:** CSE 802 — Information Security & Cryptography  
**Program:** PMICS Batch 6 | University of Dhaka | 2026  
**Submitted To:** Mukul Ahmed

[![MITRE ATT&CK](https://img.shields.io/badge/MITRE-ATT%26CK-red?style=flat-square)](https://attack.mitre.org/)
[![Splunk](https://img.shields.io/badge/Splunk-Enterprise-green?style=flat-square)](https://www.splunk.com)
[![Sysmon](https://img.shields.io/badge/Sysmon-Enabled-blue?style=flat-square)](https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon)
[![Atomic Red Team](https://img.shields.io/badge/Atomic_Red_Team-v2.3.0-orange?style=flat-square)](https://github.com/redcanaryco/atomic-red-team)

</div>

---

## 📌 Overview

This project demonstrates the setup of a complete **SOC (Security Operations Centre)** lab environment using Splunk, Sysmon, and Atomic Red Team (ART).

### The objective is to:

- Forward logs from a Windows machine to Splunk (Kali Linux)
- Monitor system activity using Sysmon
- Simulate real-world attacks using Atomic Red Team
- Perform detection engineering using Splunk SPL queries

---

## 🏗️ Lab Architecture

```
[ KALI LINUX ]                          [ WINDOWS SERVER 2019 ]
(Control & SIEM)                         (Victim & Telemetry)

+------------------+    Port 9997    +-------------------------+
|  Splunk Server   | <-------------- |  Splunk Univ. Forwarder |
|  (Indexer + Web) |                 |  Sysmon (Event Logging) |
+------------------+                 |  Atomic Red Team (ART)  |
|  Atomic Red Team | -- Attack  -->  +-------------------------+
|  (ART Scripts)   |    Emulate
+------------------+

         ISOLATED VM NETWORK (HOST-ONLY / NAT)
```

**Kali Linux** → Splunk Indexer (SIEM)

**Windows Server** → Victim Machine + Forwarder + Sysmon + ART

![Lab Architecture](images/Archi.png)

---

## 🖥️ VM Specifications

| Virtual Machine | CPU (Core) | RAM (GB) | Network Type |
|---|:---:|:---:|---|
| Kali Linux | 4 | 4 | NAT |
| Windows Server 2019 | 2 | 4 | NAT |

---

# ⚙️ 1. Splunk Universal Forwarder Installation (Windows)

## 1.1 Download Splunk Universal Forwarder

Download the Splunk Universal Forwarder (UF) from the official Splunk website: https://www.splunk.com/en_us/download/universal-forwarder.html

![Download UF](images/image1.png)

Select the correct Windows version (64-bit) and download.

![Select Version](images/image2.png)

---

## 1.2 Installation Steps

Run the installer on the Windows (victim) machine and accept the license agreement to proceed.

![Accept License](images/image3.png)

Set a local admin username and password. These credentials are used later for troubleshooting and configuration on the UF client.

![Set Credentials](images/image4.png)

---

## 1.3 Universal Forwarder Configuration (Victim Machine)

Deployment Server configuration is optional for this lab.

![Deployment Server](images/image5.png)

Configure the receiving indexer IP: `192.168.65.140` (Kali Splunk Indexer) with default port `9997`.

![Set Indexer IP](images/image6.png)

Complete the UF client installation and configuration.

![Installation Complete](images/image7.png)

---

## 1.4 Post-Installation Configuration

Navigate to:
```
C:\Program Files\SplunkUniversalForwarder\etc\system\local
```

Create or edit `inputs.conf`:

```ini
[WinEventLog://Microsoft-Windows-Sysmon/Operational]
disabled = 0
index = wineventlog
renderXml = true
```

![inputs.conf](images/image8.png)

> ⚠️ Ensure the index name (`wineventlog`) exactly matches the index created in Splunk on Kali.

---

# 🖥️ 2. Splunk Indexer Setup (Kali Linux)

## 2.1 Download & Install Splunk

Download Splunk Enterprise on Kali Linux from: https://www.splunk.com

![Download Splunk](images/image9.png)

Install Splunk and set up admin credentials for GUI access and management:

```bash
sudo dpkg -i splunk-*.deb
sudo chown -R splunk:splunk /opt/splunk
sudo /opt/splunk/bin/splunk start --accept-license
sudo /opt/splunk/bin/splunk enable boot-start
```

![Install Splunk](images/image10.png)

Verify Splunk service is running properly:

```bash
sudo /opt/splunk/bin/splunk status
```

![Splunk Running](images/image12.png)

---

## 2.2 Access Splunk Web GUI

Open a browser and navigate to `http://localhost:8000`. Log in with your configured admin credentials.

![Splunk Login](images/image13.png)

---

## 2.3 Configure Kali Indexer from GUI

**Create a New Index:**

Navigate: **Settings → Indexes → New Index**

- Index Name: `wineventlog`
- Ensure this name matches the `inputs.conf` entry on the Windows machine

![Create Index](images/image14.png)

---

## 2.4 Configure Receiving Port

Navigate: **Settings → Forwarding and Receiving → Configure Receiving → New Receiving Port**

- Port: `9997`
- Status: **Enabled**

![Configure Port 9997](images/image15.png)

Verify the port is active on Kali:

```bash
sudo ss -tlnp | grep 9997
# Expected output: 0.0.0.0:9997
```

---

# 🔍 3. Sysmon Installation (Windows)

## 3.1 Download Sysmon

Download the Sysmon executable from the official Microsoft Sysinternals page:

🔗 https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon

Extract the downloaded zip file to `C:\Tools\Sysmon`:

![Sysmon Extracted](images/image16.png)

---

## 3.2 Apply Configuration

Download the Sysmon XML configuration file from Wazuh:

🔗 https://wazuh.com/resources/blog/emulation-of-attack-techniques-and-detection-with-wazuh/sysmonconfig.xml

![Download Config](images/image17.png)

Place the XML config file in the same directory as Sysmon (`C:\Tools\Sysmon`):

![Config in Directory](images/image18.png)

---

## 3.3 Install Sysmon

Run the following command in PowerShell **as Administrator**:

```powershell
.\Sysmon64.exe -accepteula -i .\sysmonconfig.xml
```

![Sysmon Installed](images/image19.png)

Verify Sysmon is running:

```powershell
Get-Service Sysmon64
# Status: Running
```

Confirm logs in Event Viewer:
```
Applications and Services Logs > Microsoft > Windows > Sysmon > Operational
```

---

# 🔄 4. Forwarder Validation (Victim Machine)

## 4.1 Check Forwarding Status

Run the following command in the SplunkUniversalForwarder `bin` directory:

```powershell
splunk list forward-server
```

![Forward Server Check](images/image20.png)

This confirms:
- ✅ Active forwarder connection
- ✅ Destination: Kali IP (Indexer)
- ✅ Port: 9997

---

## 4.2 Verify Logs in Splunk

Open the Splunk Web GUI on Kali and run a search:

```spl
index=wineventlog
| stats count by EventCode
| sort -count
```

Logs should appear under `index=wineventlog` with Sysmon Event IDs.

![Logs Visible in Splunk](images/image21.png)

---

# ⚔️ 5. Atomic Red Team (ART) Setup on Victim Windows Machine

## 5.1 Pre-requisite — Disable Windows Defender

To run ART installations and attack simulations without interference, disable Windows Defender real-time protection. Otherwise, ART scripts and payloads will be automatically removed or quarantined.

![Disable Defender 1](images/image22.png)

![Disable Defender 2](images/image23.png)

---

## 5.2 Install Atomic Red Team

Run in **PowerShell as Administrator**:

```powershell
IEX (IWR 'https://raw.githubusercontent.com/redcanaryco/invoke-atomicredteam/master/install-atomicredteam.ps1' -UseBasicParsing)
```

Download all atomic test definitions:

```powershell
Install-AtomicRedTeam -getAtomics -Force
```

![ART Installing](images/image24.png)

![ART Atomics Downloaded](images/image25.png)

All attack configuration files are downloaded locally to `C:\AtomicRedTeam\atomics` — one folder per MITRE technique.

---

## 5.3 PowerShell Configuration

Apply the following commands one by one:

```powershell
# Enable TLS 1.2
[Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12

# Install package manager
Install-PackageProvider -Name NuGet -MinimumVersion 2.8.5.201 -Force

# Avoid confirmation prompts during installs
Set-PSRepository -Name "PSGallery" -InstallationPolicy Trusted
```

---

## 5.4 Install Required Modules

```powershell
# Fix potential module install issues
Install-Module PowerShellGet -Force -AllowClobber

# Install the main ART tool
Install-Module Invoke-AtomicRedTeam -Force

# Load commands like Invoke-AtomicTest
Import-Module Invoke-AtomicRedTeam

# Allow scripts to run (PowerShell blocks scripts by default)
Set-ExecutionPolicy Bypass -Scope Process -Force
```

---

## 5.5 Set Atomic Path

Set the environment path so every `Invoke-AtomicTest` call looks for techniques in the correct folder:

```powershell
$env:PathToAtomicsFolder = "C:\AtomicRedTeam\atomics"

$PSDefaultParameterValues = @{
    "Invoke-AtomicTest:PathToAtomicsFolder" = "C:\AtomicRedTeam\atomics"
}
```

---

## 5.6 Validate Installation

```powershell
Get-Command Invoke-AtomicTest
```

![ART Validated](images/image26.png)

> ✅ Expected version: **2.3.0**

---

## 5.7 Install Additional Modules

```powershell
Install-Module -Name AtomicTestHarnesses -Scope CurrentUser -Force
Import-Module AtomicTestHarnesses
```

This extra module provides additional helpers and dependencies required by certain atomic tests.

---

# 📊 6. Log Monitoring in Splunk

Once the full setup is complete:

- Sysmon captures all system-level events on Windows
- The Universal Forwarder ships them to Splunk on Kali in real time
- Logs can be searched, visualised, and used to build detections

**Sample test** — executing `T1059.001` to confirm end-to-end log flow:

```powershell
Invoke-AtomicTest T1059.001 -TestNumbers 6
```

![Log Monitoring](images/image27.png)

Confirm in Splunk:

```spl
index=wineventlog EventCode=1 Image="*\\powershell.exe"
| table _time, host, User, CommandLine
| sort -_time
```

---

# 🎯 7. Attack Simulation & Detection Output

> This section covers all **5 MITRE ATT&CK attack simulations**, the SPL queries used to detect each, and an explanation of which Sysmon Event ID was most helpful.
>
> 🚨 **Rule:** Detection must be based on real technical behaviour — specific Sysmon Event IDs and command-line patterns. Searching for the word "Atomic" is **not allowed**.

---

## 7.1 T1053.005 (Persistence) | Scheduled Task

**Objective:** Create a scheduled task that runs a hidden script every minute — simulating how malware establishes persistent access.

**Attack Simulation:**

I initiated the ART attack simulation (`Invoke-AtomicTest T1053.005`) on the victim Windows machine to generate relevant event logs. These logs are forwarded to the Splunk dashboard through Sysmon and the Universal Forwarder.

```powershell
# View available tests
Invoke-AtomicTest T1053.005 -ShowDetailsBrief

# Execute Test 8: Import XML Scheduled Task with Hidden Attribute
Invoke-AtomicTest T1053.005 -TestNumbers 8
```

![T1053.005 Attack 1](images/T1053_1.png)

![T1053.005 Attack 2](images/T1053_2.png)

**SPL Query:**

```spl
index=wineventlog (EventCode=4698 OR EventCode=106 OR EventCode=1 OR EventCode=4688 OR EventCode=11 OR EventCode=7)
| eval Activity = case(
    EventCode=4698, "Task Created (Security)",
    EventCode=106, "Task Registered (Task-Sched)",
    EventCode=1 OR EventCode=4688, "Process Execution",
    EventCode=11, "File Created (Sysmon)",
    EventCode=7, "Module Loaded (Sysmon)",
    1=1, "Other Activity")
| eval task_info = coalesce(TaskName, Task_Name, "N/A")
| eval user_info = coalesce(user, SubjectUserName, User, "N/A")
| eval process_info = coalesce(Image, NewProcessName, "N/A")
| eval is_suspicious=if(NOT match(task_info, "^\\\\Microsoft\\\\Windows\\\\.*") AND (match(CommandLine, ".*(powershell|cmd|temp|AppData).*") OR task_info!="N/A"), "HIGH", "LOW")
| table _time, dest, user_info, Activity, EventCode, task_info, process_info, CommandLine, is_suspicious
| sort - _time
```

**SPL Query on Splunk:**

![T1053.005 SPL](images/T1053_3.png)

**Detected Events:**

![T1053.005 Result 1](images/T1053_5.png)

![T1053.005 Result 2](images/T1053_6.png)

![T1053.005 Result 3](images/T1053_7.png)

**Explanation — Sysmon Event ID:**

**Sysmon Event ID 1** (Process Creation) is most helpful because it captures the exact `CommandLine` and parent processes where administrative tools like `cmd.exe`, `schtasks.exe`, and `reg.exe` are abused to establish persistence and elevate privileges. Specifically:

- **Registry Hijacking for UAC Bypass:** It shows `cmd.exe` executing `reg add` to modify execution paths of trusted Management Console structures (`mscfile\shell\open\command`), forcing auto-elevating binaries like `eventvwr.msc` to run unauthorised applications such as `calc.exe` with administrative rights.

- **Obfuscated PowerShell Execution:** It exposes `powershell.exe` decoding and executing fileless payloads pulled from custom registry keys (`HKCU:\SOFTWARE\ATOMIC-T1053.005`) and evaluated using `IEX`, bypassing file-based detection.

- **Scheduled Task Persistence Registration:** It captures exact `schtasks /Create` parameters including task names (`EventViewerBypass`, `ATOMIC-T1053.005`), privilege flags (`/RL HIGHEST`), and triggers (`/SC ONLOGON`, `/sc daily`).

- **Advanced Offensive Tooling:** It records execution of `PsExec.exe` spawning a SYSTEM shell and `GhostTask.exe` manipulating scheduled tasks outside of standard event logging.

---

## 7.2 T1218.005 (Defence Evasion) | MSHTA

**Objective:** Execute a malicious remote `.hta` file using `mshta.exe` to bypass application control — a classic Living off the Land (LOLBIN) technique.

**Attack Simulation:**

I initiated the ART attack simulation (`Invoke-AtomicTest T1218.005`) on the victim Windows machine to generate relevant event logs.

```powershell
# View available tests
Invoke-AtomicTest T1218.005 -ShowDetailsBrief

# Execute Test 3: MSHTA executing remote .hta payload
Invoke-AtomicTest T1218.005 -TestNumbers 3
```

![T1218.005 Attack](images/T1218_1.png)

**SPL Query:**

```spl
index=wineventlog (sourcetype="XmlWinEventLog:Microsoft-Windows-Sysmon/Operational" OR "Sysmon" OR "Microsoft-Windows-Sysmon")
(EventCode=1 OR EventID=1 OR EventCode=4688 OR EventCode=7 OR EventCode=11)
(Image="*\\mshta.exe" OR OriginalFileName="MSHTA.EXE")
(
    CommandLine="*http://*"
    OR CommandLine="*https://*"
    OR CommandLine="*.hta*"
    OR CommandLine="*vbscript:*"
    OR CommandLine="*javascript:*"
    OR CommandLine="*AppData*"
    OR CommandLine="*Temp*"
)
| eval Activity = "MSHTA Execution (T1218.005 - Defense Evasion)"
| eval Time=strftime(_time,"%Y-%m-%d %H:%M:%S")
| eval user_info=coalesce(User, user, SubjectUserName, AccountName, "N/A")
| eval process_info=coalesce(Image, ProcessName, "N/A")
| eval process_name=mvindex(split(process_info,"\\"),-1)
| eval cmdline=coalesce(CommandLine, Process_Command_Line, "N/A")
| eval parent_process=coalesce(ParentImage, ParentProcessName, ParentCommandLine, "N/A")
| eval is_suspicious=case(
    match(cmdline,"(?i)http://|https://"), "HIGH - Remote Payload Execution",
    match(cmdline,"(?i)javascript:|vbscript:"), "HIGH - Script Execution (LOLBIN Abuse)",
    match(cmdline,"(?i)\.hta"), "HIGH - HTA Payload",
    match(cmdline,"(?i)AppData|Temp|Users"), "HIGH - Suspicious Execution Path",
    parent_process!="N/A" AND NOT match(parent_process,"(?i)explorer.exe|cmd.exe|powershell.exe|winword.exe|excel.exe"), "MEDIUM - Unusual Parent Process",
    1=1, "LOW"
)
| eval mitre_technique="T1218.005 - MSHTA Defense Evasion"
| table Time host user_info Activity EventCode process_name process_info cmdline parent_process is_suspicious mitre_technique
| sort -_time
```

**SPL Query on Splunk:**

![T1218.005 SPL](images/T1218_2.png)

**Detected Events:**

![T1218.005 Result](images/T1218_3.png)

**Explanation — Sysmon Event ID:**

**Sysmon Event ID 1** (Process Creation) is most helpful because it captures the exact `cmdline` and parent processes where `mshta.exe` is abused for defence evasion. Specifically:

- **Remote Payload Execution:** It exposes `mshta.exe` reaching out to external URLs to fetch and execute remote `.hta` or `.sct` files, bypassing standard perimeter controls.

- **Inline Scripting Abuse:** It captures inline VBScript or JavaScript execution (e.g., `Wscript.Shell`, `GetObject`), allowing attackers to launch secondary payloads like PowerShell filelessly.

- **Suspicious Parent Processes:** It highlights instances where `mshta.exe` is anomalously spawned by non-standard parents like `cmd.exe`, `powershell.exe`, or `WmiPrvSE.exe`.

- **Persistence Placement:** It documents absolute file paths, flagging payloads dropped into sensitive locations like the user's `Startup` folder.

---

## 7.3 T1003.001 (Credential Access) | LSASS Dumping

**Objective:** Use `procdump` to dump the memory of `lsass.exe` and steal credentials (NTLM hashes, Kerberos tickets) from memory.

**Attack Simulation:**

I initiated the ART attack simulation (`Invoke-AtomicTest T1003.001`) on the victim Windows machine to generate relevant event logs.

> ⚠️ Temporarily disable Windows Defender real-time protection before running this test.

```powershell
# View available tests
Invoke-AtomicTest T1003.001 -ShowDetailsBrief

# Execute Test 1: LSASS dump via procdump
Invoke-AtomicTest T1003.001 -TestNumbers 1
```

![T1003.001 Attack](images/T1003_ART.png)

**SPL Query:**

```spl
index=wineventlog (
    (EventCode=10 AND TargetImage="*\\lsass.exe")
    OR
    (EventCode=11 AND TargetFilename="*.dmp")
)
| eval Activity=case(
    EventCode=10, "LSASS Access Detected (Process Access)",
    EventCode=11, "Memory Dump File Created",
    1=1, "Other Activity"
)
| eval Time=strftime(_time,"%Y-%m-%d %H:%M:%S")
| eval host=coalesce(Computer, host)
| eval user_info=coalesce(User, user, SubjectUserName, AccountName, "N/A")
| eval process_info=coalesce(SourceImage, Image, ProcessName, "N/A")
| eval process_name=mvindex(split(process_info,"\\"),-1)
| eval target_process=coalesce(TargetImage, "N/A")
| eval cmdline=coalesce(CommandLine, Process_Command_Line, "N/A")
| eval file_dump=coalesce(TargetFilename, "N/A")
| eval access_level=coalesce(GrantedAccess, "N/A")
| eval detection_type=case(
    EventCode=10 AND match(target_process,"(?i)lsass"), "LSASS Memory Access (Direct)",
    EventCode=11 AND match(file_dump,"(?i)\.dmp"), "Memory Dump File Created",
    1=1, "Suspicious Activity"
)
| eval is_suspicious=case(
    EventCode=10 AND match(access_level,"(?i)0x1fffff|0x1f3fff|0x1f0fff"), "CRITICAL - Full LSASS Access",
    EventCode=11 AND match(file_dump,"(?i)\.dmp"), "HIGH - Dump File Generated",
    match(process_info,"(?i)procdump|rundll32|taskmgr|powershell|wmic"), "HIGH - Known Dump Tool",
    1=1, "MEDIUM"
)
| eval mitre_technique="T1003.001 - LSASS Credential Dumping"
| table Time host user_info EventCode Activity process_name process_info target_process cmdline access_level file_dump detection_type is_suspicious mitre_technique
| sort - _time
```

**SPL Query on Splunk:**

![T1003.001 SPL](images/T1003_2.png)

**Detected Events:**

![T1003.001 Result](images/T1003_1.png)

**Explanation — Sysmon Event ID:**

**Sysmon Event IDs 10** (Process Access) **and 11** (File Create) are most helpful because they capture memory access and dumping behaviours targeting `lsass.exe`. Specifically:

- **Direct Memory Access Tracking:** Event ID 10 exposes source processes (`powershell.exe`, `rundll32.exe`, `rdrleakdiag.exe`) requesting handles to read the virtual memory of `lsass.exe`.

- **Highly Suspicious Access Masks:** It logs precise `GrantedAccess` values — such as `0x1FFFFF` (Full Access) or `0x1F3FFF` — which are strong indicators of NTLM hash or Kerberos ticket harvesting.

- **LOLBIN Identification:** It highlights non-standard applications accessing LSASS memory, tracking both scripting hosts (`powershell.exe`) and diagnostic tools (`rdrleakdiag.exe`) acting as dump tools.

- **Physical Dump File Creation:** Event ID 11 links memory access to disk activity, recording the exact `.dmp` file path (e.g., `...\AppData\Local\Temp\lsass-comsvcs.dmp`) when `rundll32.exe` invokes `comsvcs.dll` to write credentials to disk.

---

## 7.4 T1059.001 (Execution) | PowerShell Download

**Objective:** Use PowerShell to download and execute a script from the web — the standard technique used in malware dropper and fileless attack stages.

**Attack Simulation:**

I initiated the ART attack simulation (`Invoke-AtomicTest T1059.001`) on the victim Windows machine to generate relevant event logs.

```powershell
# View available tests
Invoke-AtomicTest T1059.001 -ShowDetailsBrief

# Execute Test 6: Pure PowerShell download and execute
Invoke-AtomicTest T1059.001 -TestNumbers 6
```

![T1059.001 Attack](images/T1059_1.png)

**SPL Query:**

```spl
index=wineventlog
(EventCode=1 OR EventCode=4688 OR EventID=1)
(Image="*\\powershell.exe" OR Image="*\\pwsh.exe" OR NewProcessName="*\\powershell.exe" OR NewProcessName="*\\pwsh.exe")
(CommandLine="*Invoke-WebRequest*" OR CommandLine="*IEX*" OR CommandLine="*Invoke-Expression*"
 OR CommandLine="*DownloadString*" OR CommandLine="*Net.WebClient*"
 OR CommandLine="*Start-BitsTransfer*" OR CommandLine="*curl*" OR CommandLine="*wget*"
 OR CommandLine="*http://*" OR CommandLine="*https://*"
 OR CommandLine="*-enc*" OR CommandLine="*EncodedCommand*")
| eval Activity="PowerShell Execution - Download / Payload Staging (T1059.001)"
| eval Time=strftime(_time,"%Y-%m-%d %H:%M:%S")
| eval host=coalesce(host, Computer)
| eval user_info=coalesce(User, user, SubjectUserName, AccountName, "N/A")
| eval process_info=coalesce(Image, NewProcessName, ProcessName)
| eval process_name=mvindex(split(process_info,"\\"),-1)
| eval cmdline=coalesce(CommandLine, Process_Command_Line, "N/A")
| eval parent_process=coalesce(ParentImage, ParentProcessName)
| eval event_source=case(
    EventCode=1 OR EventID=1, "Sysmon",
    EventCode=4688, "Windows Security",
    1=1, "Unknown"
)
| eval technique="T1059.001 - PowerShell Execution"
| eval is_suspicious=case(
    match(cmdline,"(?i)Invoke-WebRequest|DownloadString|Net.WebClient|Start-BitsTransfer"), "HIGH - File Download via PowerShell",
    match(cmdline,"(?i)IEX|Invoke-Expression"), "CRITICAL - Code Injection Execution",
    match(cmdline,"(?i)-enc|EncodedCommand"), "CRITICAL - Obfuscation",
    match(cmdline,"(?i)http://|https://"), "HIGH - Remote URL Execution",
    1=1, "MEDIUM"
)
| table Time user_info EventCode process_name technique parent_process is_suspicious cmdline
| sort - _time
```

**SPL Query on Splunk:**

![T1059.001 SPL](images/T1059_3.png)

**Detected Events:**

![T1059.001 Result](images/T1059_2.png)

**Explanation — Sysmon Event ID:**

**Sysmon Event ID 1** (Process Creation) is most helpful because it captures the exact `cmdline` and parent processes where `powershell.exe` is abused for download and execution. Specifically:

- **Remote Script Download and Execution:** It exposes PowerShell using `DownloadString`, `Invoke-WebRequest`, or `Msxml2.ServerXMLHTTP` to fetch payloads from external repositories like GitHub (e.g., `Invoke-Mimikatz.ps1`).

- **Advanced Command Obfuscation:** It captures heavily obfuscated, Base64-encoded command parameters (`-EncodedArguments`) alongside complex environment variable variations designed to evade detection.

- **In-Memory Payload Loading:** It logs sensitive cmdlets (`PowerUp`, `PowerView`, `Get-Keystrokes`, `Add-Persistence`) passed directly into memory, avoiding disk-based antivirus detection entirely.

- **Suspicious Shell Spawning:** It documents abnormal behaviour where `powershell.exe` is recursively spawned by another PowerShell instance or triggered remotely via `WmiPrvSE.exe`.

---

## 7.5 T1112 (Defence Evasion) | Registry Modification

**Objective:** Disable Windows Defender by modifying registry keys — blinding the victim's security tools before further attacks.

**Attack Simulation:**

I initiated the ART attack simulation (`Invoke-AtomicTest T1112`) on the victim Windows machine to generate relevant event logs.

```powershell
# View available tests
Invoke-AtomicTest T1112 -ShowDetailsBrief

# Test 38: Suppress Windows Defender Notifications
Invoke-AtomicTest T1112 -TestNumbers 38

# Test 51: Disable Windows Defender Notifications
Invoke-AtomicTest T1112 -TestNumbers 51

# Test 56: Tamper with Defender Protection (may be partially blocked)
Invoke-AtomicTest T1112 -TestNumbers 56
```

> Tests 38 and 51 completed successfully. Test 56 was partially blocked by Windows Security, but notifications were already suppressed by the previous two tests.

![T1112 Attack](images/T1112_1.png)

**SPL Query:**

```spl
index=wineventlog (sourcetype="XmlWinEventLog:Microsoft-Windows-Sysmon/Operational" OR "Sysmon" OR "Microsoft-Windows-Sysmon")
(EventCode=1 OR EventID=1 OR EventCode=4688 OR EventCode=7 OR EventCode=11)
(
    Image="*\\reg.exe"
    OR Image="*\\powershell.exe"
    OR Image="*\\pwsh.exe"
    OR Image="*\\cmd.exe"
)
(
    CommandLine="*Windows Defender*"
    OR CommandLine="*DisableAntiSpyware*"
    OR CommandLine="*TamperProtection*"
    OR CommandLine="*DisableNotifications*"
    OR CommandLine="*Notification_Suppress*"
    OR CommandLine="*DisableRealtimeMonitoring*"
    OR CommandLine="*DisableBehaviorMonitoring*"
    OR CommandLine="*DisableOnAccessProtection*"
    OR CommandLine="*Set-MpPreference*"
    OR CommandLine="*Add-MpPreference*"
    OR CommandLine="*MpPreference*"
    OR CommandLine="*Defender*"
)
| eval Activity="Registry / Defender Tampering (T1112 - Defense Evasion)"
| eval Time=strftime(_time,"%Y-%m-%d %H:%M:%S")
| eval host=coalesce(host, Computer)
| eval user_info=coalesce(User, user, SubjectUserName, AccountName, "N/A")
| eval process_info=coalesce(Image, ProcessName)
| eval process_name=mvindex(split(process_info,"\\"),-1)
| eval cmdline=coalesce(CommandLine, Process_Command_Line, "N/A")
| eval parent_process=coalesce(ParentImage, ParentProcessName, ParentCommandLine, "N/A")
| eval technique="T1112 - Modify Registry"
| eval detection_type=case(
    match(cmdline,"(?i)Set-MpPreference|Add-MpPreference"), "CRITICAL - Defender Policy Modified",
    match(cmdline,"(?i)DisableAntiSpyware|TamperProtection"), "CRITICAL - Defender Disabled",
    match(cmdline,"(?i)DisableRealtimeMonitoring|DisableBehaviorMonitoring"), "HIGH - Real-time Protection Disabled",
    match(cmdline,"(?i)Windows Defender"), "HIGH - Defender Configuration Access",
    match(process_info,"(?i)reg.exe|powershell|pwsh"), "HIGH - Registry Modification Tool",
    1=1, "MEDIUM"
)
| eval is_suspicious=case(
    match(cmdline,"(?i)Set-MpPreference|DisableAntiSpyware|TamperProtection"), "CRITICAL",
    match(cmdline,"(?i)DisableRealtimeMonitoring|DisableBehaviorMonitoring"), "HIGH",
    1=1, "MEDIUM"
)
| table Time host user_info EventCode EventID process_name process_info parent_process cmdline detection_type is_suspicious Activity technique
| sort - _time
```

**SPL Query on Splunk:**

![T1112 SPL](images/T1112_3.png)

**Detected Events:**

![T1112 Result](images/T1112_2.png)

**Explanation — Sysmon Event ID:**

**Sysmon Event ID 1** (Process Creation) is most helpful because it captures the exact `cmdline` and process details where system tools are abused to modify registry keys for defence evasion. Specifically:

- **Security Feature Disabling:** It highlights `reg.exe` or `cmd.exe` directly turning off core antivirus protections, such as setting `TamperProtection` to `0` under `Windows Defender\Features`.

- **Alert and Notification Suppression:** It logs commands targeting `Windows Defender\UX Configuration` and `Windows Security Center\Notifications` to set `Notification_Suppress` or `DisableNotifications` values to `1`.

- **Telemetry and Reporting Blindness:** It captures registry entries aimed at turning off security health visibility, specifically changes to `Windows Defender\Reporting` implementing `DisableEnhancedNotifications`.

- **Indirect Execution Chain:** It records parent-child process relationships, catching instances where `powershell.exe` spawns `cmd.exe` as a middleman to issue the malicious `reg add` configurations.

---

## 📊 Detection Summary Table

| # | MITRE ID | Technique | Tactic | Key Event ID | Why This Event ID? |
|:---:|---|---|---|:---:|---|
| 1 | **T1053.005** | Scheduled Task | Persistence | **EID 1** | Captures `schtasks.exe`/PowerShell with task-creation command lines, privilege flags, XML task import |
| 2 | **T1218.005** | MSHTA | Defense Evasion | **EID 1** | Captures `mshta.exe` with remote URLs, inline VBScript/JS, anomalous parent processes |
| 3 | **T1003.001** | LSASS Dumping | Credential Access | **EID 10 + 11** | EID 10: memory handle on `lsass.exe`; EID 11: `.dmp` file written to disk |
| 4 | **T1059.001** | PowerShell Download | Execution | **EID 1** | Captures PowerShell with download + execute patterns (`DownloadString`, `IEX`, `-enc`) |
| 5 | **T1112** | Registry Modification | Defense Evasion | **EID 1** | Captures `reg.exe`/PowerShell targeting Defender registry keys with exact command lines |

---

## 🔐 Final Notes

> All attack simulations were conducted in a **controlled, isolated lab environment for educational purposes only.**

<div align="center">

🛡️ Defense &nbsp;|&nbsp; 🔎 Detection &nbsp;|&nbsp; ⚔️ Simulation &nbsp;|&nbsp; 📊 Analysis

**CSE 802: Information Security & Cryptography | PMICS Batch 6 | University of Dhaka | 2026**

</div>
