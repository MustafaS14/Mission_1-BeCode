# Detection: Mission 01, Step 6

Once everyone's machine was connected, the coach ran something on all our workstations at the same time, without telling us what it was or when it would happen. This is what I found in my SIEM.

## How I went looking

First I checked that my machine was still sending heartbeats during that time. Otherwise "nothing found" could just mean "nothing collected". Then I went through `SecurityEvent` around the time it happened, looking for anything that didn't match my own activity: event IDs I hadn't seen before, accounts I don't use, logons I didn't make.

What actually led me to it was the workstation itself. `Get-LocalUser` showed an account I hadn't seen in Know your seat, `svc_backup`, and the local Security log had a 4720 for it. My first try in Sentinel was to filter on an Event ID I already expected, which doesn't make sense when the Event ID is what you're looking for. So instead I listed every account-management event in the last 30 days. Only one burst came back, all in the same second.

## The query

```kql
SecurityEvent
| where TimeGenerated > ago(30d)
| where EventID in (4720, 4722, 4724, 4725, 4726, 4728, 4732, 4738, 4781)
| project TimeGenerated, EventID, Activity, SubjectAccount, TargetAccount, MemberSid, Computer
| order by TimeGenerated asc
```

## What it returned

All on 9 October 2026, on WKS-L58.hamilton.corp, done by `HAMILTON\WKS-L58$`:

| Time (UTC) | Event ID | Target | What it shows |
|---|---|---|---|
| 12:30:07.127 | 4728 | WKS-L58\None | The new account's SID (…-1008) added to the local "None" group, which Windows does for every new local user |
| 12:30:07.128 | 4720 | WKS-L58\svc_backup | A user account was created |
| 12:30:07.131 | 4722 | WKS-L58\svc_backup | The account was enabled |
| 12:30:07.131 | 4738 | WKS-L58\svc_backup | The account was changed |
| 12:30:07.167 | 4738 | WKS-L58\svc_backup | The account was changed again |
| 12:30:07.167 | 4724 | WKS-L58\svc_backup | Its password was reset |
| 12:30:07.207 | 4732 | Builtin\Administrators | SID …-1008 (svc_backup) added to the local Administrators group |

The flags for the CTF were `HAM{4720}` (creation) and `HAM{4732}` (promotion).

## What I think happened

Someone created a local account, `svc_backup`, gave it a password and put it in the local Administrators group, all within 80 milliseconds. No person types that fast, so it was a script. The account doing it is `HAMILTON\WKS-L58$`, the computer's own account, which is how actions running as SYSTEM show up. That fits the "What just happened?" part of the CTF: the coach drives our VMs from the hypervisor through the QEMU Guest Agent (`qemu-ga`), which runs as SYSTEM, without opening a session.

The name is also a typical trick: `svc_backup` looks like a service account, the kind nobody questions, but it's a brand-new local admin.

## Why I know it wasn't me

- My only RDP logon that day was at 09:27:44 UTC (logon type 7, from the lab gateway 10.50.0.1), three hours earlier.
- Between 11:30 and 13:30 UTC there's no logon of type 10 or 7 at all, only the machine account's network logons (type 3) and SYSTEM's service logons (type 5).
- The events are signed by the computer account, not by `HAMILTON\mustafa`.

## Where the SIEM was blind

The same intrusion also changed my domain account: it was added to the domain group `L58-Project-Readers`. That change was logged on DC01, the domain controller, not on my workstation. My workspace only receives logs from WKS-L58, so none of it shows in Sentinel. I could only see it from the workstation with `net user %USERNAME% /domain` after signing out and in again.

## What I'd look at next

- Whether `svc_backup` has logged on since (4624 with `TargetUserName == "svc_backup"`), and from where.
- An analytics rule in Sentinel that alerts on 4720 followed by 4732 for the same SID within a few minutes.
- Turning on process creation auditing (4688) or installing Sysmon, so I can see which program ran the commands, not only their result.
- Collecting the Application log at Information level, or at least the `qemu-ga` source, so guest agent activity reaches the SIEM.
- Connecting DC01 (or getting its logs from the coach) so changes to domain accounts are visible too.
