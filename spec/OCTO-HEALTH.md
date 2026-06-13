# OCTO Health Layer
## Status Surface, Arm Monitoring, and Practice Reporting

OCTO runs silently. Arms reach, surfaces appear, actions complete.
But the practice needs to know it's working — at a glance,
without thinking about it, without opening a dashboard.

The health layer is the ambient signal that OCTO is alive,
arms are reporting, and nothing is silently failing.

Three things. Always visible. Never intrusive.

```
1. OCTO is running          →  not crashed, not hung
2. Arms are healthy         →  each source reachable and reporting
3. Daily history at a glance →  how did today go, any failures
```

---

## The Health Signal Vocabulary

### Per Arm

| Signal | Meaning |
|---|---|
| 🟢 Healthy | Responded within expected time, data returned |
| 🟡 Degraded | Slow response, empty result, or partial data |
| 🔴 Failed | Error, timeout, or connection refused |
| ⚪ Skipped | Not scheduled for this run |
| 🔵 Running | Currently executing |

### Overall OCTO

| Signal | Meaning |
|---|---|
| ● Running | Process alive, arms reporting |
| ○ Idle | Between scheduled runs, all healthy |
| ⚠ Attention | One or more arms degraded |
| ✕ Error | Run failed — check log |

---

## Output Layer 1 — Windows System Tray Icon

Lightest possible surface. Lives in the bottom-right corner.
Color signals health. Click for detail. Zero screen real estate.

```
🟢  All arms healthy — last run successful
🟡  One arm slow or returned empty
🔴  Arm failed — needs attention
⚪  OCTO not running
```

**Right-click tray menu:**

```
OCTO  ●  Running
─────────────────────────────
Last run: today 08:47  ✓
─────────────────────────────
View morning brief
View arm health
Run now
─────────────────────────────
Settings
Exit
```

This is what the dental receptionist sees.
They're not monitoring infrastructure.
They just need to know it worked.
Green means good. Start the day.

---

## Output Layer 2 — OCTO Health Panel

A small floating window. Minimizes to tray.
Shows arm status live. Stays out of the way.

```
┌─────────────────────────────────────┐
│  OCTO  ●  Running          08:47   │
├─────────────────────────────────────┤
│                                     │
│  Arms                    Status     │
│  ─────────────────────────────────  │
│  outlook.inbox           🟢  47ms  │
│  outlook.calendar        🟢  23ms  │
│  teams.planner           🟡  slow  │
│  git.commits             🟢  12ms  │
│                                     │
│  Last run    08:47  ✓  success     │
│  Next run    17:00  scheduled      │
│                                     │
│  Today       3 runs  0 failures    │
│                                     │
│  [Run now]  [View brief]  [─]  [×] │
└─────────────────────────────────────┘
```

Small. Floats over other windows.
The receptionist glances at it the way you glance at a clock.
Double-click the tray icon to show or hide.

---

## Output Layer 3 — OCTO Daily Report

End of day summary. Appears at 5pm or on request.
Full picture of how OCTO performed today.

```
┌──────────────────────────────────────────┐
│  OCTO Daily Report — Tuesday June 17    │
├──────────────────────────────────────────┤
│                                          │
│  Runs today          4                  │
│  Successful          4                  │
│  Failed              0                  │
│  Actions taken       7                  │
│                                          │
│  Arms performance                        │
│  outlook.inbox    ████████████  healthy │
│  outlook.calendar ████████████  healthy │
│  teams.planner    ████████░░░░  slow ×2 │
│  git.commits      ████████████  healthy │
│                                          │
│  Actions completed today                 │
│  ✓  Confirmation emails sent    3       │
│  ✓  Waitlist slot filled        1       │
│  ✓  Insurance pre-auth drafted  2       │
│  ✓  Morning brief delivered     1       │
│                                          │
│  Arm issue detected                      │
│  teams.planner responded slowly twice   │
│  Suggestion: check Teams connectivity   │
│                                          │
│  [Dismiss]  [Save report]  [Send to me] │
└──────────────────────────────────────────┘
```

Appears automatically at end of day.
Dismissed in one click.
Saved or emailed if the owner wants a record.

---

## Output Layer 4 — Weekly Summary Email

Every Monday morning. Practice owner's inbox.
The value proof. The retention mechanism.

```
Subject: OCTO Weekly Summary — week of June 15

OCTO ran reliably this week.

  Runs completed     28 of 28
  Actions taken      94
  Time saved est.    ~8 hours

  Confirmations sent        34  (3 cancellations caught early)
  Reminders delivered       28
  Waitlist slots filled      4
  Insurance pre-auths        6

  One arm was slow Tuesday afternoon — Teams Planner.
  Resolved itself by Wednesday. No action needed.

  Next week: 67 appointments scheduled.
  OCTO will brief your team at 8:45 each morning.
```

That email is why they never cancel.
Every Monday they see the value in concrete numbers.
Hours saved. Actions taken. Problems caught early.

Not a feature. Proof.

---

## Who Sees What

```
Dental receptionist   →  Tray icon — green means good
                          Morning brief surface at 8:45
                          Daily report at 5pm if something was slow

Practice owner        →  Weekly summary email every Monday
                          Concrete numbers: time saved, actions taken
                          Arm issues flagged before they become problems

Configurator (you)    →  Full arm logs if something needs fixing
                          Remote diagnostic reach if needed
                          state.json and history folder — full picture
```

---

## Implementation — Single C# File

The health monitor runs as a lightweight background process.
Reads state files. Renders tray icon and health panel on demand.
No always-on agent. No token consumption while idle.

```csharp
#:package System.Windows.Forms@6.0.0
#:package System.Text.Json@8.0.0

using System.Windows.Forms;
using System.Text.Json;

// Load state written by OCTO runs
var armStatus = LoadArmStatus(@"C:\reach\state\arms.json");
var runHistory = LoadRunHistory(@"C:\reach\history");

// Tray icon — simplest viable output
var trayIcon = new NotifyIcon
{
    Icon = GetHealthIcon(armStatus.OverallHealth),
    Visible = true,
    Text = $"OCTO — Last run: {runHistory.LastRun:HH:mm} " +
           $"{(runHistory.LastRunSuccess ? "✓" : "✕")}"
};

// Double-click → show health panel
trayIcon.DoubleClick += (s, e) =>
    ShowHealthPanel(armStatus, runHistory);

// Right-click → tray menu
trayIcon.ContextMenuStrip = BuildTrayMenu(armStatus, runHistory);

// Lightweight Windows message loop — near-zero CPU
Application.Run();

// Health icon based on overall arm state
Icon GetHealthIcon(HealthState state) => state switch
{
    HealthState.Healthy   => LoadIcon("green"),
    HealthState.Degraded  => LoadIcon("yellow"),
    HealthState.Failed    => LoadIcon("red"),
    _                     => LoadIcon("grey")
};
```

---

## Arm State File — `arms.json`

Written by every OCTO run. Read by the health monitor.
Simple JSON. Human readable. Git trackable.

```json
{
  "lastUpdated": "2026-06-17T08:47:23Z",
  "overallHealth": "healthy",
  "lastRunSuccess": true,
  "lastRunAt": "2026-06-17T08:47:00Z",
  "nextRunAt": "2026-06-17T17:00:00Z",
  "todayRuns": 3,
  "todayFailures": 0,
  "arms": [
    {
      "name": "outlook.inbox",
      "status": "healthy",
      "responseMs": 47,
      "lastResult": "47 items scanned",
      "lastRunAt": "2026-06-17T08:47:01Z"
    },
    {
      "name": "outlook.calendar",
      "status": "healthy",
      "responseMs": 23,
      "lastResult": "12 appointments today",
      "lastRunAt": "2026-06-17T08:47:02Z"
    },
    {
      "name": "teams.planner",
      "status": "degraded",
      "responseMs": 4200,
      "lastResult": "partial — 3 of 8 boards loaded",
      "lastRunAt": "2026-06-17T08:47:06Z",
      "note": "slow response — Teams connectivity may be degraded"
    }
  ]
}
```

---

## Run History — `history/` Folder

Every OCTO run writes a `.reach-artifact`.
Health monitor reads the folder to build the daily report.

```
C:\reach\history\
  2026-06-17-0847-morning-brief.reach-artifact
  2026-06-17-1200-midday-check.reach-artifact
  2026-06-17-1700-daily-report.reach-artifact
  2026-06-16-0847-morning-brief.reach-artifact
  ...
```

Each artifact contains:
- Run timestamp and duration
- Arms that ran and their results
- Actions taken
- Any errors or degraded states
- The surface that was shown

The health monitor reads these to build:
- Today's run count and failure count
- Weekly summary totals
- Arm performance trends
- Anomaly detection — "Teams was slow 3 times this week"

---

## Anomaly Detection — When to Surface Issues

The health layer doesn't alert on every hiccup.
It surfaces patterns worth attention.

```
Alert immediately:
  Arm failed 2+ consecutive runs      →  tray icon red, notification
  OCTO process not running at schedule →  notification "OCTO didn't run"
  Critical arm down (inbox, calendar)  →  immediate tray notification

Surface in daily report:
  Arm slow 2+ times today             →  flagged with suggestion
  Run took 2× longer than usual       →  noted, no alarm
  Empty result where data expected    →  flagged for review

Surface in weekly summary:
  Arm degraded 3+ times this week    →  trend flagged to owner
  Actions count dropped vs last week  →  noted
  New pattern detected                →  "Mondays are busiest — consider earlier run"
```

---

## Dental Practice — Full Health Picture

What the dental practice sees across a typical day:

```
08:44  Task Scheduler wakes morning.octo
08:45  Arms reach simultaneously
08:47  Morning brief surface appears at receptionist desk
08:47  Tray icon: 🟢 all arms healthy
08:48  Receptionist clicks [Send confirmations]
08:49  3 confirmation emails sent via Outlook
08:49  OCTO closes: "Good start. 12 patients today."
08:49  Tray icon: 🟢 running / idle

12:00  Midday check runs — schedule changes, cancellations
12:01  Tray icon: 🟢 healthy
12:02  1 cancellation detected → surface appears
       "Johnson 2pm cancelled — waitlist slot available"
       [Fill slot]  [Leave open]  [Dismiss]
12:03  Receptionist clicks [Fill slot]
12:03  Waitlist patient contacted via Outlook draft

17:00  Daily report appears
17:00  Tray icon: 🟢 4 runs, 0 failures, 7 actions
17:01  Receptionist dismisses — day done

Mon    Practice owner receives weekly summary email
       "28 runs. 94 actions. ~8 hours saved."
```

Nothing breaks. Nothing needs monitoring.
The tray is green. The coffee is hot.

---

## The Configurator View — Remote Diagnostic

If something does need attention — you can reach in remotely:

```csharp
// diagnostic-remote.cs — run from your machine
// reads their state files via shared folder or VPN

reach file.read "\\their-machine\reach\state\arms.json"
  output json

reach file.read "\\their-machine\reach\history\*.reach-artifact"
  since yesterday
  output summary
```

Claude reads the state, identifies the issue, authors a fix.
You send them a new `.cs` file or updated `.octo` arm.
They drop it in the folder. Done.

No remote desktop needed for most issues.
No support ticket system.
No vendor escalation path.

---

## The Practice Kit — Complete Picture

```
OCTO Practice Kit
─────────────────────────────────────────────────────

Delivered:
  Diagnostic reach         →  understands their workflow
  Configured .octo files   →  authored for their practice
  Morning brief surface    →  the daily interaction layer
  Tray health indicator    →  ambient status, always visible
  Health panel             →  arm detail on demand
  Daily report             →  end of day summary
  Weekly summary email     →  owner sees value every Monday
  Remote diagnostic path   →  you can help without site visit

What they interact with daily:
  Morning brief surface    →  8:45am, 90 seconds, done
  Tray icon                →  glance, green, continue
  Daily report             →  5pm, one click, dismissed

What they never see:
  NuGet packages
  C# code
  COM Interop calls
  Playwright instances
  State files
  Log files
  Claude API calls

What they pay:
  Anthropic API            →  ~$5-20/month, direct to Anthropic
  Your setup fee           →  one time
  Optional retainer        →  for additions and adjustments
```

---

## Design Constraints

```
✓  Tray icon is always visible — ambient, not intrusive
✓  Health panel opens on demand — never forced
✓  Daily report appears once — dismissed in one click
✓  Weekly email is the owner's proof of value
✓  Remote diagnostic — no site visit for most issues
✓  All state in flat files — no database to maintain
✓  Near-zero CPU when idle — lightweight background process
✗  No always-on monitoring agent — no continuous token burn
✗  No cloud dashboard — everything local, everything private
✗  No app to update — files don't have versions
✗  No support ticket system — flat files are self-explanatory
```

---

## Notes

- Tray icon implemented via `System.Windows.Forms.NotifyIcon`
  built into Windows — zero additional dependencies
- Health panel uses WinForms — same as OCTO prompt surfaces
  consistent visual language throughout
- State files written by every OCTO run — health monitor
  reads them, never writes them — clean separation
- Weekly summary email sent via Outlook COM reach —
  same arm that handles all email actions
- Patient data never leaves the machine — Outlook COM
  is local, no cloud processing of sensitive information
  this is a feature, not a limitation — more private
  than any SaaS practice management alternative
