# Detection — Mission 01, Step 6

> **Status: empty template.** Step 6 hasn't happened yet, or I haven't written up what I found. Every `TODO` must come from my own SIEM. Don't guess.

## What I was told
The coach ran something on every workstation at once. We weren't told what it was or when it would start. My job was to find it in my SIEM and show it.

## How I looked for it
1. Checked `Heartbeat` first to make sure my machine was still reporting during the window.
2. Looked for anything unusual in `SecurityEvent` around the time of the session: event IDs I don't normally see, accounts I don't recognise, logons from unexpected places.
3. `TODO: describe the steps I actually took and what led me to the event.`

## The query
```kql
// TODO: the exact query that shows the event
SecurityEvent
| where TimeGenerated > ago(TODO)
| where EventID == TODO
| project TimeGenerated, Computer, Account, EventID, Activity, LogonType, IpAddress
| order by TimeGenerated desc
```

## The result
| TimeGenerated (UTC) | Computer | EventID | Account | Details |
|---|---|---|---|---|
| `TODO` | `TODO` | `TODO` | `TODO` | `TODO` |

(Optional: add a screenshot of the result.)

## What I think happened
`TODO: in my own words. What was done, by which account, when, and how.`

## How I know it wasn't me
`TODO: e.g. account I don't use, time I wasn't connected, logon type or source address that doesn't match my RDP sessions (mine come from 10.50.0.1, logon type 10/7).`

## Open questions
`TODO: anything I couldn't explain, or what I would check next.`
