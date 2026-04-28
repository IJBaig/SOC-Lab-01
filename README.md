# SOC Lab (Splunk UF + WinEventLog + Dashboard)

**Date:** 2026-04-28
---

## 1) Overview (What you built)
The goate is to create a SOC Detection lab from scrach. A basic Lab that will show alerts about login failure, sccessful logins, sysmon Process images,  on windows syst
### Architecture
- **Windows 10 host**
  - Runs **Splunk Universal Forwarder**
  - Collects:
    - Windows Event Logs (Security/System/Application)
    - (Optional) Sysmon channel
  - Forwards to Kali on TCP **9997**
- **Kali Linux (Splunk Enterprise)**
  - Acts as **receiver/indexer**
  - Stores indexed data locally and provides Splunk Web UI on **8000**

### Data Flow
Windows Event Log / Sysmon → Splunk UF → TCP 9997 → Splunk Receiver on Kali → Index (`win10`) → Splunk searches + dashboard

---

## 2) Prerequisites

### Windows 10
- Local admin access
- Splunk Universal Forwarder installed
- Network connectivity to Kali IP (example: `192.168.56.1`)
- Recommended: enough disk space for UF logs and Windows Event Logs

### Kali Linux
- Splunk Enterprise installed in `/opt/splunk`
- Stable IP reachable from Windows (example: `192.168.56.1`)
- Port **9997** accessible (no firewall blocking; or rule allowing it)
- Splunk Web reachable on **8000** (locally or via tunnel)

### Network / Ports
- Kali Splunk Web: `8000/tcp`
- Splunk management: `8089/tcp` (used internally; useful for troubleshooting)
- Splunk receiving: `9997/tcp` (Windows UF sends events here)

---

## 3) Installation & Configuration (Step-by-step)

### 3.1 Kali: Enable Splunk receiver on port 9997
On Kali:

1. Start Splunk (if not already running):
   - `sudo /opt/splunk/bin/splunk start`

2. Enable receiver:
   - `sudo /opt/splunk/bin/splunk enable listen 9997`

3. Restart Splunk (recommended):
   - `sudo /opt/splunk/bin/splunk restart`

4. Verify Splunk is listening on 9997:
   - `sudo ss -lntp | grep 9997`

Expected: `splunkd` listening on `0.0.0.0:9997` (or Kali’s IP).

---

### 3.2 Kali: Create/verify the index (`win10`)
In Splunk Web:
1. Go to **Settings → Indexes**
2. Create index:
   - Name: `win10`
   - Retention: default is fine for lab
3. Save

Verification search:
- `index=win10 | head 5`

---

### 3.3 Windows: Configure Splunk Universal Forwarder to send to Kali
On Windows (Admin CMD):

1. Verify the forward-server config:
   - `splunk list forward-server`

2. If not configured, add forward server:
   - `splunk add forward-server 192.168.56.1:9997`

3. Verify it becomes active (after receiver is listening):
   - `splunk list forward-server`

Expected:
- **Active forwards:** `192.168.56.1:9997`

---

### 3.4 Windows: Configure WinEventLog collection (inputs.conf)
Edit/create:

`C:\Program Files\SplunkUniversalForwarder\etc\system\local\inputs.conf`

Add stanzas (example baseline):

- Application log
- Security log
- System log
- (Optional) Sysmon log

Restart UF after changes:
- `sc stop SplunkForwarder`
- `sc start SplunkForwarder`

**Important note:** If you want events to land in `index=win10`, you must set `index = win10` inside each stanza.

---

### 3.5 Windows: Ensure UF starts on boot
UF runs as a Windows service: **SplunkForwarder**

1. Confirm service exists:
   - `sc query SplunkForwarder`

2. Ensure auto start:
   - `sc config SplunkForwarder start= auto`

3. Start service:
   - `sc start SplunkForwarder`

4. Verify:
   - `sc qc SplunkForwarder`  
   Confirm `START_TYPE : 2 AUTO_START`

---

### 3.6 Sysmon (Optional but Recommended)
Sysmon provides high-signal telemetry (process creation, network connections, etc.) that improves SOC detections.

**Install:**
1. Run Sysmon as Administrator:
   - `sysmon64.exe -accepteula -i`

2. Verify Sysmon service:
   - `sc query Sysmon64` (should be RUNNING)

**Collect Sysmon in Splunk UF:**
Ensure `inputs.conf` includes:
- `WinEventLog://Microsoft-Windows-Sysmon/Operational`

Restart UF:
- `sc stop SplunkForwarder`
- `sc start SplunkForwarder`

**Validate Sysmon is generating logs locally:**
- Check Event Viewer → Sysmon Operational log
- Or use PowerShell:
  - `Get-WinEvent -LogName "Microsoft-Windows-Sysmon/Operational" -MaxEvents 5`

---

## 4) Validation & Troubleshooting

### 4.1 Verify forwarder is connected (Kali)
On Kali:
- `sudo /opt/splunk/bin/splunk list forward-server`

You should see the Windows forwarder listed as connected.

### 4.2 Verify data exists in Splunk (Web)
In Splunk Web (Search & Reporting), set time to **All time** and run:
- `index=* | stats count by index | sort -count`

If you see data in `main` but not `win10`, it usually means:
- `index = win10` missing in `inputs.conf`, or
- data was ingested before you created/used `win10`, or
- another config overrides it.

### 4.3 Validate specific Windows data is coming in
- `index=win10 sourcetype=WinEventLog:Security | head 20`
- `index=win10 sourcetype=WinEventLog:System | head 20`
- `index=win10 sourcetype=WinEventLog:Application | head 20`

### 4.4 If UF shows Active but Splunk shows no events
Common checks:
- Receiver enabled + listening on 9997 on Kali
- Time range in Web (use All time)
- Index permissions for the user in Splunk Web
- Verify `inputs.conf` is actually being used (btool debug on Windows):
  - `splunk btool inputs list --debug`

### 4.5 Log locations (for diagnostics)
**On Windows (UF internal logs):**
- `C:\Program Files\SplunkUniversalForwarder\var\log\splunk\splunkd.log`

**On Kali (Splunk internal logs):**
- `/opt/splunk/var/log/splunk/splunkd.log`

---

## 5) Core SOC Searches (Commands to Use)

> Replace `index=win10` if you used another index name.

### 5.1 Data inventory
**Count by sourcetype:**
- `index=win10 earliest=-24h | stats count by sourcetype | sort -count`

**Count by host and sourcetype:**
- `index=win10 earliest=-24h | stats count by host sourcetype | sort -count`

### 5.2 Authentication monitoring
**Failed logons (4625):**
- `index=win10 sourcetype=WinEventLog:Security EventCode=4625 earliest=-24h | stats count by Account_Name Source_Network_Address host | sort -count`

**Successful logons (4624):**
- `index=win10 sourcetype=WinEventLog:Security EventCode=4624 earliest=-24h | stats count by Account_Name Logon_Type host | sort -count`

### 5.3 Account / admin changes (high-signal)
**New local user created (4720):**
- `index=win10 sourcetype=WinEventLog:Security EventCode=4720 earliest=-7d | table _time host SubjectUserName TargetUserName Message`

### 5.4 Process creation visibility
If Security process creation is enabled (4688):
- `index=win10 sourcetype=WinEventLog:Security EventCode=4688 earliest=-24h | table _time host Account_Name New_Process_Name Process_Command_Line Parent_Process_Name | head 50`

### 5.5 Sysmon (if enabled)
**Sysmon process creation (Event ID 1):**
- `index=win10 sourcetype="WinEventLog:Microsoft-Windows-Sysmon/Operational" EventCode=1 earliest=-24h | table _time host Image CommandLine ParentImage User | head 50`

**Sysmon network connections (Event ID 3):**
- `index=win10 sourcetype="WinEventLog:Microsoft-Windows-Sysmon/Operational" EventCode=3 earliest=-24h | stats count by DestinationIp Image | sort -count | head 20`

### 5.6 PowerShell hunting (basic)
Works best with Sysmon Event ID 1 or Security 4688:
- `index=win10 earliest=-24h (powershell OR pwsh) | table _time host sourcetype EventCode Message | head 50`

---

## 6) Dashboard (Final Step)

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

## 7) Operational Notes / Best Practices (Lab)

- Keep Splunk Web restricted; avoid exposing it publicly unless you secure it properly.
- Use a dedicated index (`win10`) and apply retention suitable for lab storage.
- If performance warnings appear (slow configuration initialization), it usually indicates low disk/RAM; reduce apps/add-ons and ensure sufficient resources.
- Sysmon significantly improves detection quality—recommended for SOC-style dashboards and alerts.

---

## 8) Quick Checklist (Success Criteria)

- [ ] Kali: receiver enabled on **9997**
- [ ] Windows: UF configured and shows **Active forward**
- [ ] Splunk Web: `index=win10` returns events
- [ ] Dashboard panels populate (at least Security/System/Application)
- [ ] (Optional) Sysmon panels populate after Sysmon is installed + forwarded
