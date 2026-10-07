# Runbook — Mission 01: Getting eyes on my perimeter

> **Status: first draft.** Anything marked `TODO` is something only I can fill in, like real names, times, error messages and what actually happened. Check every step against what I really did before handing it in.

## Goal

Connect a Windows 10 lab workstation to my own Microsoft Sentinel workspace, check that logs are arriving, and use it to find an event I did not cause (Step 6).

## What I started with

- A Windows 10 workstation on the lab network, reached by RDP over Tailscale
- My own Azure for Students subscription (BeCode school account)
- My access sheet: workstation address, login, first password

## Names and settings I used

| Item | Value |
|---|---|
| Azure region (used everywhere) | `TODO` |
| Resource group | `rg-sentinel-lab` |
| Log Analytics workspace | `log-sentinel-lab` |
| DCR (Azure Monitor, chain A) | `dcr-windowsevents` |
| DCR (Sentinel connector, chain B) | `dcr-securityevents` |
| Arc machine name | `TODO (e.g. WKS-Lxx)` |
| Daily cap | 0.2 GB/day |

---

## Build order

### 0. Get on the lab network (Tailscale)
1. Opened Tailscale on my laptop and signed in with the account my coach gave me.
2. Waited for the coach to approve my device.
3. `ping <workstation-address>` until I got replies. **Do not open RDP before the ping works.**

What broke: `TODO (or "nothing")`

### 1. Reach the machine (RDP)
1. Connected with `TODO (mstsc / Windows App / Remmina)` using the address and account from my sheet.
2. Changed my password at first logon, as expected.

What broke: `TODO`

### 2. Build the SIEM
1. **Azure for Students:** portal → *Education* → *Sign up now* → *Start free*. Used my **BeCode school account** (top-right of the portal must show BECODE). Country: Belgium. Address: BeCentral, Cantersteen 15, 1000 Brussels.
   Check: *Education → Overview* shows 100 USD / 365 days.
2. **Allowed region:** portal → *Policy → Assignments → Allowed resource deployment regions*. Picked `TODO`, because `TODO (EU? offered for RG, workspace and Arc?)`.
3. **Resource group** `rg-sentinel-lab` → **Log Analytics workspace** `log-sentinel-lab` (Pay-as-you-go, Per GB 2018) → **Microsoft Sentinel** on that workspace. The 31-day free trial started at that point.
4. **Daily cap:** workspace → *Settings → Usage and estimated costs → Daily cap* → On, 0.2 GB/day.
   The trade-off: when the cap is reached, collection stops until the next day, so the cap is a safety net and should sit above normal volume.

What broke: `TODO (e.g. RequestDisallowedByAzure because of the region policy?)`

### 3. Connect the machine
**3.1 Azure Arc**
1. Portal → *Azure Arc → Machines → Onboard/Create → Onboard existing machines*.
2. Settings: same RG and region · Windows · **untick Connect SQL Server** · Public endpoint · Authenticate machines manually · nothing paid enabled under *Management*.
3. On the workstation, PowerShell **as administrator**:
   ```powershell
   Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass -Force
   ```
   then pasted the Arc script.
4. Signed in when the browser opened inside the VM.
5. *Azure Arc → Machines* → refreshed until the machine showed **Connected**.

What broke and how I fixed it: `TODO`. These are the traps the mission warns about. Keep the ones that happened to me and delete the rest:
- **The script was blocked by the execution policy.** Fix: run the `Set-ExecutionPolicy` line in the same admin window, then run the script again.
- **`(400) Bad Request` from `Invoke-WebRequest`.** This is only the failed error report, not the real problem. Fix: check admin rights, the execution policy and the region.
- **`RequestDisallowedByAzure` / 401 about MFA.** The agent installed, but the browser sign-in inside the VM had no MFA. Fix: in the same window I ran
  ```powershell
  & "$env:ProgramW6432\AzureConnectedMachineAgent\azcmagent.exe" connect --resource-group "$env:RESOURCE_GROUP" --tenant-id "$env:TENANT_ID" --location "$env:LOCATION" --subscription-id "$env:SUBSCRIPTION_ID" --cloud "$env:CLOUD" --tags 'ArcSQLServerExtensionDeployment=Disabled' --use-device-code
  ```
  and entered the code at `https://login.microsoft.com/device` **from my laptop**.

**⚠️ My mistake here (this actually happened to me):**
- **What I did:** the `--use-device-code` command printed a link (`https://login.microsoft.com/device`) and a code. I opened the link and entered the code **in the browser on the remote workstation (inside my RDP session)**, not on my own laptop.
- **What happened:** the sign-in still came from the VM's browser, which never asks for MFA. Azure refused to create the machine again, with the same MFA error. `TODO: confirm the exact message I saw`
- **Why:** the device-code step exists so the sign-in happens in a browser that **is** signed in with MFA, which is the one on my laptop where I'm already logged into the Azure portal. Doing it inside the VM changes nothing.
- **Fix:** ran the same `azcmagent connect ... --use-device-code` command again in the same admin PowerShell window. Then I opened `https://login.microsoft.com/device` **on my laptop**, entered the new code and signed in. The window ended with *Machine connected to Azure*.
- **Lesson for the next person:** when PowerShell on the workstation gives you a link and a code, **copy them to your own laptop**. Don't click the link inside the RDP session.

**3.2 Data Collection Rule (chain A)**
1. *Monitor → Settings → Data Collection Rules → + Create* (from Azure Monitor, **not** Sentinel).
2. Name `dcr-windowsevents`, same RG and region, *Agent-based – Windows or Linux*, no DCE, no managed identity.
3. Resources: only my Arc machine.
4. Data source: Windows Event Logs (Basic). Application and System: Critical, Error, Warning. Security: Audit success and Audit failure. Information and Verbose left unticked because they are too noisy.
5. Destination: Log Analytics → `log-sentinel-lab`. The data lands in the **`Event`** table.
6. Check: *Arc → machine → Extensions* shows **AzureMonitorWindowsAgent = Succeeded** (after about `TODO` minutes).

Warning: do **not** install the `AzureMonitorAgentClientSetup.msi`. The agent comes in through Arc.

What broke: `TODO`

### 4. Prove it is alive
In *Sentinel → Logs* (switch the editor to **KQL mode**), I ran `Heartbeat | take 10`. My machine name showed up after `TODO` minutes.

Reflexes:
- **Check `Heartbeat` first.** If it is missing, the problem is the connection, not the logs.
- **Don't check the agent from the machine.** `Get-Service AzureMonitorAgent` returns nothing on an Arc machine. The SIEM is the place to check.

**Event Viewer exercise.** In the Security log, filtered on 4624, I found my own logon:
| Time | Account (New Logon) | Logon Type | Source network address |
|---|---|---|---|
| `TODO` | `TODO` | `TODO (10 = new RDP, 7 = reconnect)` | `TODO` |

### 5. First questions
I ran queries 1–4 (see `queries.kql`). Query 2 compares `TimeGenerated` with `ingestion_time()`. It tells "the source is dead" apart from "the source's clock is wrong". The delay I saw: `TODO`.

### 5b. The SOC way (chain B)
1. *Sentinel → Content hub* → installed **Windows Security Events**. If you get "page moved to Defender portal", do a hard reload (`Ctrl+F5` / `Cmd+Shift+R`).
2. *Data connectors* → **Windows Security Events via AMA**, not the `[DEPRECATED]` legacy one → *Open connector page* → *+ Create data collection rule*.
3. Name `dcr-securityevents`, same RG, my Arc machine, events: **Common** (the default is *All*, so I changed it).
4. After about `TODO` minutes, query 5 returned `SecurityEvent` rows with `Account`, `LogonType` and `IpAddress` as columns.
5. **Removed the duplicate:** opened `dcr-windowsevents` → Windows Event Logs → unticked both Security boxes → Save at `TODO (time)`.
   Check: about 15 minutes later, the newest row of query 4 (`Event`) was still older than that time, while query 5 (`SecurityEvent`) kept getting new rows.

Things that surprised me: one RDP connection gives several 4624 events in the same second (type 3 for NLA, then 10 or 7). `IpAddress` shows `10.50.0.1`, which is the lab gateway, not my laptop.

What broke: `TODO`

---

## Chain A vs chain B

| | Chain A: Azure Monitor DCR → `Event` | Chain B: Sentinel connector → `SecurityEvent` |
|---|---|---|
| Where it is set up | Azure Monitor (works without Sentinel) | Inside Sentinel (*Windows Security Events* solution) |
| What it collects | Any Windows log (Application, System, Security…) | Security log only |
| How events are picked | By log and level (or XPath) | Preset sets: All / Common / Minimal / Custom |
| Shape of the data | One text field (`RenderedDescription`), so I have to parse it myself | Already split into columns (`Account`, `LogonType`, `IpAddress`…) |
| Sentinel rules and workbooks | Mostly not written for it | Written for it |
| Typical users | IT operations | Security teams / SOC |
| Cost | Per GB | Per GB. With both on, every security event is paid twice |

**Why I kept only chain B for the Security log** (`TODO: rewrite this in my own words`):
With both chains on, the same logon is stored twice, in two tables. That means two places to search for one fact, two answers that can drift apart, and paying twice for the same data. Chain B gives the fields an analyst needs already in columns, and Sentinel's detections, workbooks and the next missions are built on `SecurityEvent`. Chain A stays useful for what B can't collect: the Application and System logs.

---

## If I had to rebuild this
`TODO: the 3–5 things I'd tell the next person.` Starting points:
- Choose the region from the policy list first, then use it everywhere.
- Before running the Arc script: admin PowerShell plus the `Set-ExecutionPolicy` line.
- Enter the device-login code on your own laptop, never in the browser inside the VM.
- Give the agent 15–30 minutes. Check `Heartbeat` before assuming anything is broken.
# Runbook — Mission 01: Getting eyes on my perimeter

> **Status: first draft.** Anything marked `TODO` is something only I can fill in, like real names, times, error messages and what actually happened. Check every step against what I really did before handing it in.

## Goal

Connect a Windows 10 lab workstation to my own Microsoft Sentinel workspace, check that logs are arriving, and use it to find an event I did not cause (Step 6).

## What I started with

- A Windows 10 workstation on the lab network, reached by RDP over Tailscale
- My own Azure for Students subscription (BeCode school account)
- My access sheet: workstation address, login, first password

## Names and settings I used

| Item | Value |
|---|---|
| Azure region (used everywhere) | `TODO` |
| Resource group | `rg-sentinel-lab` |
| Log Analytics workspace | `log-sentinel-lab` |
| DCR (Azure Monitor, chain A) | `dcr-windowsevents` |
| DCR (Sentinel connector, chain B) | `dcr-securityevents` |
| Arc machine name | `TODO (e.g. WKS-Lxx)` |
| Daily cap | 0.2 GB/day |

---

## Build order

### 0. Get on the lab network (Tailscale)
1. Opened Tailscale on my laptop and signed in with the account my coach gave me.
2. Waited for the coach to approve my device.
3. `ping <workstation-address>` until I got replies. **Do not open RDP before the ping works.**

What broke: `TODO (or "nothing")`

### 1. Reach the machine (RDP)
1. Connected with `TODO (mstsc / Windows App / Remmina)` using the address and account from my sheet.
2. Changed my password at first logon, as expected.

What broke: `TODO`

### 2. Build the SIEM
1. **Azure for Students:** portal → *Education* → *Sign up now* → *Start free*. Used my **BeCode school account** (top-right of the portal must show BECODE). Country: Belgium. Address: BeCentral, Cantersteen 15, 1000 Brussels.
   Check: *Education → Overview* shows 100 USD / 365 days.
2. **Allowed region:** portal → *Policy → Assignments → Allowed resource deployment regions*. Picked `TODO`, because `TODO (EU? offered for RG, workspace and Arc?)`.
3. **Resource group** `rg-sentinel-lab` → **Log Analytics workspace** `log-sentinel-lab` (Pay-as-you-go, Per GB 2018) → **Microsoft Sentinel** on that workspace. The 31-day free trial started at that point.
4. **Daily cap:** workspace → *Settings → Usage and estimated costs → Daily cap* → On, 0.2 GB/day.
   The trade-off: when the cap is reached, collection stops until the next day, so the cap is a safety net and should sit above normal volume.

What broke: `TODO (e.g. RequestDisallowedByAzure because of the region policy?)`

### 3. Connect the machine
**3.1 Azure Arc**
1. Portal → *Azure Arc → Machines → Onboard/Create → Onboard existing machines*.
2. Settings: same RG and region · Windows · **untick Connect SQL Server** · Public endpoint · Authenticate machines manually · nothing paid enabled under *Management*.
3. On the workstation, PowerShell **as administrator**:
   ```powershell
   Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass -Force
   ```
   then pasted the Arc script.
4. Signed in when the browser opened inside the VM.
5. *Azure Arc → Machines* → refreshed until the machine showed **Connected**.

What broke and how I fixed it: `TODO`. These are the traps the mission warns about. Keep the ones that happened to me and delete the rest:
- **The script was blocked by the execution policy.** Fix: run the `Set-ExecutionPolicy` line in the same admin window, then run the script again.
- **`(400) Bad Request` from `Invoke-WebRequest`.** This is only the failed error report, not the real problem. Fix: check admin rights, the execution policy and the region.
- **`RequestDisallowedByAzure` / 401 about MFA.** The agent installed, but the browser sign-in inside the VM had no MFA. Fix: in the same window I ran
  ```powershell
  & "$env:ProgramW6432\AzureConnectedMachineAgent\azcmagent.exe" connect --resource-group "$env:RESOURCE_GROUP" --tenant-id "$env:TENANT_ID" --location "$env:LOCATION" --subscription-id "$env:SUBSCRIPTION_ID" --cloud "$env:CLOUD" --tags 'ArcSQLServerExtensionDeployment=Disabled' --use-device-code
  ```
  and entered the code at `https://login.microsoft.com/device` **from my laptop**.

**3.2 Data Collection Rule (chain A)**
1. *Monitor → Settings → Data Collection Rules → + Create* (from Azure Monitor, **not** Sentinel).
2. Name `dcr-windowsevents`, same RG and region, *Agent-based – Windows or Linux*, no DCE, no managed identity.
3. Resources: only my Arc machine.
4. Data source: Windows Event Logs (Basic). Application and System: Critical, Error, Warning. Security: Audit success and Audit failure. Information and Verbose left unticked because they are too noisy.
5. Destination: Log Analytics → `log-sentinel-lab`. The data lands in the **`Event`** table.
6. Check: *Arc → machine → Extensions* shows **AzureMonitorWindowsAgent = Succeeded** (after about `TODO` minutes).

Warning: do **not** install the `AzureMonitorAgentClientSetup.msi`. The agent comes in through Arc.

What broke: `TODO`

### 4. Prove it is alive
In *Sentinel → Logs* (switch the editor to **KQL mode**), I ran `Heartbeat | take 10`. My machine name showed up after `TODO` minutes.

Reflexes:
- **Check `Heartbeat` first.** If it is missing, the problem is the connection, not the logs.
- **Don't check the agent from the machine.** `Get-Service AzureMonitorAgent` returns nothing on an Arc machine. The SIEM is the place to check.

**Event Viewer exercise.** In the Security log, filtered on 4624, I found my own logon:
| Time | Account (New Logon) | Logon Type | Source network address |
|---|---|---|---|
| `TODO` | `TODO` | `TODO (10 = new RDP, 7 = reconnect)` | `TODO` |

### 5. First questions
I ran queries 1–4 (see `queries.kql`). Query 2 compares `TimeGenerated` with `ingestion_time()`. It tells "the source is dead" apart from "the source's clock is wrong". The delay I saw: `TODO`.

### 5b. The SOC way (chain B)
1. *Sentinel → Content hub* → installed **Windows Security Events**. If you get "page moved to Defender portal", do a hard reload (`Ctrl+F5` / `Cmd+Shift+R`).
2. *Data connectors* → **Windows Security Events via AMA**, not the `[DEPRECATED]` legacy one → *Open connector page* → *+ Create data collection rule*.
3. Name `dcr-securityevents`, same RG, my Arc machine, events: **Common** (the default is *All*, so I changed it).
4. After about `TODO` minutes, query 5 returned `SecurityEvent` rows with `Account`, `LogonType` and `IpAddress` as columns.
5. **Removed the duplicate:** opened `dcr-windowsevents` → Windows Event Logs → unticked both Security boxes → Save at `TODO (time)`.
   Check: about 15 minutes later, the newest row of query 4 (`Event`) was still older than that time, while query 5 (`SecurityEvent`) kept getting new rows.

Things that surprised me: one RDP connection gives several 4624 events in the same second (type 3 for NLA, then 10 or 7). `IpAddress` shows `10.50.0.1`, which is the lab gateway, not my laptop.

What broke: `TODO`

---

## Chain A vs chain B

| | Chain A: Azure Monitor DCR → `Event` | Chain B: Sentinel connector → `SecurityEvent` |
|---|---|---|
| Where it is set up | Azure Monitor (works without Sentinel) | Inside Sentinel (*Windows Security Events* solution) |
| What it collects | Any Windows log (Application, System, Security…) | Security log only |
| How events are picked | By log and level (or XPath) | Preset sets: All / Common / Minimal / Custom |
| Shape of the data | One text field (`RenderedDescription`), so I have to parse it myself | Already split into columns (`Account`, `LogonType`, `IpAddress`…) |
| Sentinel rules and workbooks | Mostly not written for it | Written for it |
| Typical users | IT operations | Security teams / SOC |
| Cost | Per GB | Per GB. With both on, every security event is paid twice |

**Why I kept only chain B for the Security log** (`TODO: rewrite this in my own words`):
With both chains on, the same logon is stored twice, in two tables. That means two places to search for one fact, two answers that can drift apart, and paying twice for the same data. Chain B gives the fields an analyst needs already in columns, and Sentinel's detections, workbooks and the next missions are built on `SecurityEvent`. Chain A stays useful for what B can't collect: the Application and System logs.

---

## If I had to rebuild this
`TODO: the 3–5 things I'd tell the next person.` Starting points:
- Choose the region from the policy list first, then use it everywhere.
- Before running the Arc script: admin PowerShell plus the `Set-ExecutionPolicy` line.
- Give the agent 15–30 minutes. Check `Heartbeat` before assuming anything is broken.

