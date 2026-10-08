# Runbook: Mission 01

These are my notes from connecting my lab workstation to my own Sentinel workspace. I wrote them so someone else could rebuild the same setup from scratch, mistakes included. Anything still marked TOD[...]

## What I ended up with

| | |
|---|---|
| Region (used for everything) | francecentral |
| Resource group | rg-sentinel-lab |
| Log Analytics workspace | log-sentinel-lab (Sentinel enabled on it) |
| Daily cap | 0.2 GB/day |
| Workstation in Arc | WKS-L58 (WKS-L58.hamilton.corp) |
| DCR from Azure Monitor (chain A) | dcr-windowsevents: Application + System only now |
| DCR from the Sentinel connector (chain B) | dcr-securityevents: Security log, "Common" set |

I named everything type-first (rg-, log-, dcr-) so I can tell what something is from its name alone.

---

## How I built it, in order

### 1. Getting onto the workstation

The workstation isn't on the internet, so I first connected to the lab network with Tailscale on my laptop and waited for the coach to approve my device. Before trying RDP I pinged the workstation add[...]

Once the ping worked I connected with TODO (RDP client) and changed the password at first login.

### 2. Setting up the Azure side

I activated Azure for Students with my BeCode school account. Signing in with a personal Microsoft account here leads to an empty subscription. I checked the Education page showed the 100 USD credit b[...]

Next I had to pick a region. Our student subscriptions only allow a few regions, and the list isn't the same for everyone. I found mine under Policy → Assignments → "Allowed resource deployment re[...]

Then I created, in this order:
1. the resource group,
2. the Log Analytics workspace (Pay-as-you-go pricing),
3. Microsoft Sentinel on top of the workspace. This starts a 31-day free trial.

Last, I set a daily cap of 0.2 GB on the workspace (Usage and estimated costs → Daily cap). Our credit is limited, and one noisy log source could use a lot of it overnight. The cap is a safety net, [...]

### 3. Connecting the workstation with Azure Arc

Azure can't see a machine that lives on our lab network, so the first job is to register it with Azure Arc. I started from Azure Arc → Machines → Onboard existing machines and used these settings:[...]

On the workstation I opened PowerShell as administrator. Windows 10 blocks downloaded scripts by default, so before pasting the Arc script I ran this:

```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass -Force
```

It only affects that one window.

The script installed the agent and opened a browser inside the VM for me to sign in. The agent installed, but Azure refused to create the machine because the sign-in hadn't used MFA, and the browser i[...]

```powershell
& "$env:ProgramW6432\AzureConnectedMachineAgent\azcmagent.exe" connect --resource-group "$env:RESOURCE_GROUP" --tenant-id "$env:TENANT_ID" --location "$env:LOCATION" --subscription-id "$env:SUBSCRIPTI[...]
```

This prints a link and a code.

**This is where I made a mistake.** I opened the link and entered the code in the browser inside the VM, through my RDP session. That is the same browser without MFA, so it failed the same way (TODO: [...]

**Note for whoever does this next:** when PowerShell on the workstation gives you a link and a code, type them on your own laptop, not inside the RDP session.

TODO: other problems I hit at this step, if any.

### 4. Choosing what to collect (chain A)

Arc makes the machine visible to Azure, but nothing is sent until a data collection rule says what to collect and where to send it. I created dcr-windowsevents from Azure Monitor → Data Collection R[...]
- resource: only my Arc machine
- Windows Event Logs: Application and System at Critical, Error and Warning, plus the Security log (audit success and failure). I left Information off because it's far too noisy.
- destination: my workspace. Everything from this rule lands in the `Event` table.

Attaching the rule is what installs the Azure Monitor Agent on the machine as an Arc extension. I didn't download any installer myself. It showed as Succeeded under the machine's Extensions after abou[...]

### 5. Checking that data arrives

In Sentinel → Logs (switched to KQL mode) I ran `Heartbeat | take 10` until my machine showed up. The first heartbeat arrived on 7 October 2026 at 14:16 UTC, about 4 minutes after I created `dcr-win[...]

To find my own RDP logon, I searched `SecurityEvent` for successful logons (4624) with logon type 10 or 7, the two types an RDP session produces, and took the earliest one (query in `queries.kql`):

| Time | Logon type | Source address |
|---|---|---|
| 7 Oct 2026, 23:16:47 UTC | 7 | 10.50.0.1 |

Logon type 7 means I reconnected to a session that was already open. A brand-new RDP session would show as 10. The address is the lab gateway that relays my connection, not my laptop.

Then I started asking questions with KQL (all in `queries.kql`): when the machine last checked in, whether its data is current (event time vs ingestion time, to catch a wrong clock), what levels of ev[...]

### 6. Moving the Security log to the Sentinel connector (chain B)

Next I collected the Security log the way Sentinel expects it. In Sentinel I installed the Windows Security Events solution from the Content hub. Then I opened the **Windows Security Events via AMA** [...]

It needed another TODO minutes before rows showed up in `SecurityEvent`, but there `Account`, `LogonType` and `IpAddress` are separate columns. The first rows I got (7 October 2026, around 15:08 UTC) [...]

At that point every security event was being collected twice, once per chain. So I edited dcr-windowsevents and unticked the two Security boxes, at 15:09 UTC. About 15 minutes later, `Event` had no Se[...]

---

## Chain A vs chain B

| | Chain A: Azure Monitor DCR → `Event` | Chain B: Sentinel connector → `SecurityEvent` |
|---|---|---|
| Set up from | Azure Monitor, works without Sentinel | Sentinel, needs the Windows Security Events solution |
| Collects | Any Windows log | Only the Security log |
| How you choose events | By log and severity level | Preset sets (All / Common / Minimal / Custom) |
| What the data looks like | One text blob per event, you parse it yourself | Already split into columns |
| Sentinel's built-in rules and workbooks | Mostly don't use it | Built for it |
| Who usually uses it | IT operations | Security teams |
| Cost | Per GB | Per GB, so running both means paying twice for security events |

**Why I kept only chain B for the Security log** (TODO: check this sounds like me):
Running both meant every logon was stored twice in two different tables. That's two places to look for the same thing, results that can drift apart, and double the cost. Chain B gives me the fields I [...]

---

## If I did it again

- Pick the region from the policy list before creating anything.
- Use an admin PowerShell window and set the execution policy before running the Arc script.
- Do the device-code sign-in on your own laptop, not in the VM.
- Give the agent 15–30 minutes and check Heartbeat before deciding something's broken.
- TODO: anything else
