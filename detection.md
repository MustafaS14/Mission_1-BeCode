# Detection: Mission 01, Step 6

Once everyone's machine was connected, the coach ran something on all our workstations at the same time, without telling us what it was or when it would happen. This is what I found in my SIEM.

(TODO: everything below has to come from my own results. Fill it in after Step 6.)

## How I went looking

First I checked that my machine was still sending heartbeats during that time. Otherwise "nothing found" could just mean "nothing collected". Then I went through `SecurityEvent` around the time it happened, looking for anything that didn't match my own activity: event IDs I hadn't seen before, accounts I don't use, logons I didn't make.

TODO: what actually led me to it

## The query

```kql
// TODO: the exact query I used
SecurityEvent
| where TimeGenerated > ago(TODO)
| where EventID == TODO
| project TimeGenerated, Computer, Account, EventID, Activity, LogonType, IpAddress
| order by TimeGenerated desc
```

## What it returned

| Time (UTC) | Computer | Event ID | Account | What it shows |
|---|---|---|---|---|
| TODO | TODO | TODO | TODO | TODO |

TODO: screenshot of the result

## What I think happened

TODO

## Why I know it wasn't me

TODO. Things to compare against: my own RDP logons come in from 10.50.0.1 with logon type 10 or 7, at times I was connected.

## What I'd look at next

TODO
