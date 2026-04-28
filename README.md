# SOC Lab (Splunk UF + WinEventLog + Dashboard)

**Date:** 2026-04-28
---

## Overview (What you built)
The goate is to create a SOC Detection lab from scrach. A basic Lab that will show alerts about login failure, sccessful logins, sysmon Process images and powershell execution on windows system.

### Architecture
- **Windows 10**
  - Runs **Splunk Universal Forwarder**
- **Kali Linux (Splunk Enterprise)**
  - Acts as **receiver/indexer**
    
 Kali linux machine will act as host that runs Splunk Enterprise and The window 10 machine in virtual will run Splunk forwarder and Sysmon that forward the logs to splunk indxer on kali machine. 

### Data Flow
Windows Event Log / Sysmon → Splunk UF → TCP 9997 → Splunk Receiver on Kali → Index (`win10`) → Splunk searches + dashboard

---

## Prerequisites

  #### 1: Host Machine
- with minimum 8G ram and 50G+ Storage (15GB will also be enough)
- We will use Linux Machine (kali Linux).
 
#### 2: Virtualization Software
- we will use Virtual Box
- you can use any other but keep the configuration specially network configurations same 

#### 3: Client Machine
- we will use windows 10
 - so download the windows 10 iso from official microsoft page
---
## Installation & Configuration (Step-by-step)

### 1: Splunk Enterprise + Forwarder
- Download the Latest Splunk EnterPrise and Forwarder From Official Splunk Page
  - **Forwarder:** [link](https://www.splunk.com/en_us/download/universal-forwarder.html)
    - Forwarder will be used on client so download for specific OS ie., Windows 10 64bit
  - **Enterprise:** [link](https://www.splunk.com/en_us/download/splunk-enterprise.html)
    - Forwarder will be used on client so download for specific OS ie., Linux 64 bit (.deb file)
### 2: Install Splunk on Kali Linux
- ```bash
    dpkg -i splunk-10.2.2-80b90d638de6-linux-amd64.deb
  ```
    - Install it to /opt
    - Check the official Splunk [Blog](https://help.splunk.com/en/splunk-enterprise/get-started/install-and-upgrade/10.2/welcome-to-the-splunk-enterprise-installation-manual/whats-in-this-manual) for installation step
    - set username and password

### 3: Setup Windows 10 in virtual Box
Create a windows 10 virtual machine based on specified options
- Ram minimum 3GB
- Storage 50GB+ Dynamic allocated
- 2 Network Adapters
  - 1 NAT not NAT Network
  - 2 Host-only Adapter
    - Create a Host-Only Network Adapter From Network Tab in virtual box Homepage
      - create -> Configure Adapter Manually
        - set IP `192.168.56.1`
        - set subnet `255.255.255.0`
      - DHCP Server
        - Enable Server
        - Server Address `192.168.56.100`
        - Subnet `255.255.255.0`
        - Lower Bound Address `192.168.56.101`
        - Upper Address Bound `192.168.56.254`
- Start The Windows machine 

### 4: Install and Run Sysmon And Forwarder
#### Sysmon
Sysmon provides high-signal telemetry (process creation, network connections, etc.) that improves SOC detections.
- Download Sysmon from [Microsoft](https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon) official sysinternals.
- **unzip** it and open the **CMD** as Administrator in same Folder
  - 1. Run The Command 
     ```bash
     sysmon64.exe -accepteula -i
     ```
  - 2. Verify Sysmon service:
   - ```bash
     sc query Sysmon64
     ```
- Now Sysmon is Added in startup  
#### Splunk Forwarder
- Move Forwarder .msi to Windows 10
- click and install
  - Just set username and Password and Click continue. we wil set the rest later (we can set it rightnow but lets just do it via files it is fun)
  -  goto This Location
    - `C:\Program Files\SplunkUniversalForwarder\etc\system\local\`
    - Move the Files [input.cong](./input.conf) and [output.conf](./output.conf) here.
  - open CMD as Administrator
    - ```bash
      cd C:\Program Files\SplunkUniversalForwarder\bin
      splunk enable boot-start
      splunk restart
      splunk list forward-server
      ```
    - enter username and password for forwarder
    - it will show inactive server `192.168.1.56.1:9997`
      
### 5 Kali: Enable Splunk receiver on port 9997
On Kali:

1. Start Splunk:
   - ```bash sudo /opt/splunk/bin/splunk start```
2. Enable receiver:
   - ```bash sudo /opt/splunk/bin/splunk enable listen 9997```

3. Restart Splunk (recommended):
   - ```bash sudo /opt/splunk/bin/splunk restart```

4. Verify Splunk is listening on 9997:
   - ```bash sudo ss -lntp | grep 9997```

Expected: `splunkd` listening on `0.0.0.0:9997` (or Kali’s IP).

---

### 6 Kali: Create/verify the index (`win10`)
In Splunk Web:
1. Go to **Settings → Indexes**
2. Create index:
   - Name: `win10`
   - Retention: default is fine for lab
3. Save

Verification search:
- `index=win10 | head 5`

---

### 7 Windows: Configure Splunk Universal Forwarder to send to Kali
On Windows (Admin CMD):

1. Verify the forward-server config:
   - ```bash splunk list forward-server```

Expected:
- **Active forwards:** `192.168.56.1:9997`

## Data inventory

**Count by sourcetype:**
- `index=win10 earliest=-24h | stats count by sourcetype | sort -count`

**Count by host and sourcetype:**
- `index=win10 earliest=-24h | stats count by host sourcetype | sort -count`

**Failed logons (4625):**
- `index=win10 sourcetype=WinEventLog:Security EventCode=4625 earliest=-24h | stats count by Account_Name Source_Network_Address host | sort -count`

**Successful logons (4624):**
- `index=win10 sourcetype=WinEventLog:Security EventCode=4624 earliest=-24h | stats count by Account_Name Logon_Type host | sort -count`

**New local user created (4720):**
- `index=win10 sourcetype=WinEventLog:Security EventCode=4720 earliest=-7d | table _time host SubjectUserName TargetUserName Message`

**If Security process creation is enabled (4688):**
- `index=win10 sourcetype=WinEventLog:Security EventCode=4688 earliest=-24h | table _time host Account_Name New_Process_Name Process_Command_Line Parent_Process_Name | head 50`

**Sysmon process creation (Event ID 1):**
- `index=win10 sourcetype="WinEventLog:Microsoft-Windows-Sysmon/Operational" EventCode=1 earliest=-24h | table _time host Image CommandLine ParentImage User | head 50`

**Sysmon network connections (Event ID 3):**
- `index=win10 sourcetype="WinEventLog:Microsoft-Windows-Sysmon/Operational" EventCode=3 earliest=-24h | stats count by DestinationIp Image | sort -count | head 20`

**Works best with Sysmon Event ID 1 or Security 4688:**
- `index=win10 earliest=-24h (powershell OR pwsh) | table _time host sourcetype EventCode Message | head 50`

---

## Dashboard (Final Step)

### 6.1 Dashboard goal
A single SOC overview dashboard for Windows telemetry:
- Total events
- Hosts
- Failed logons (4625)
- Successful logons (4624)
- Events over time
- Top sourcetypes
- Top failed logon source IPs
- Top users logging in
- (Optional) Sysmon panels:
  - top processes
  - top destinations
  - PowerShell executions

### 6.2 Create the dashboard
In Splunk Web:
1. Open **Dashboards**
2. **Create New Dashboard**
3. Choose Classic or Dashboard Studio (your preference)
4. Add panels using searches from Section 5
5. **Placeholder:** *(Add your dashboard source here)*

---
## 8) Quick Checklist (Success Criteria)

- [ ] Kali: receiver enabled on **9997**
- [ ] Windows: UF configured and shows **Active forward**
- [ ] Splunk Web: `index=win10` returns events
- [ ] Dashboard panels populate (at least Security/System/Application)
- [ ] (Optional) Sysmon panels populate after Sysmon is installed + forwarded
