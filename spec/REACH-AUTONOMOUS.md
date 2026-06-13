# REACH-AUTONOMOUS — Autonomous and Multi-Agent Patterns

REACH in autonomous mode operates without a human in the loop. Claude authors the
orchestration logic, schedules it via Windows Task Scheduler, and the system wakes,
acts, records, signals, and sleeps — on its own cadence.

This document covers autonomous patterns, design principles, failure handling, and
the architectural decisions that make unattended REACH reliable.

---

## The Core Autonomous Loop

```
Wake (Task Scheduler trigger)
  ↓
Check state — what needs attention?
  ↓
Reach into sources (Outlook, DB, git, web)
  ↓
Act (send, record, launch, signal)
  ↓
Write state — what did I do?
  ↓
Log — structured, reconstructable
  ↓
Sleep (process exits, scheduler owns the next wake)
```

Every autonomous reach follows this shape. The discipline is in the state and log
layers — without them, unattended runs are a black box.

---

## Scheduling — Windows Task Scheduler

No daemon, no always-on process. Task Scheduler wakes the reach, it runs, it exits.
Energy efficient, no memory leak risk, no zombie processes.

```powershell
# Register a scheduled reach — runs every morning at 8am
$action = New-ScheduledTaskAction -Execute "dotnet" `
    -Argument "run C:\reach\outlook-check.cs" `
    -WorkingDirectory "C:\reach"

$trigger = New-ScheduledTaskTrigger -Daily -At "8:00AM"

Register-ScheduledTask -TaskName "REACH-OutlookCheck" `
    -Action $action -Trigger $trigger -RunLevel Highest
```

**Trigger types available:**
- Daily / weekly at fixed time
- On system startup
- On user login
- On file system event (via wrapper)
- On network availability
- Chained — one task signals the next

---

## State Management — Nothing Is Remembered Without Design

Each reach starts fresh. Autonomous runs need explicit state or they repeat work,
miss things, or lose track of where they left off.

**Simple state file pattern:**

```csharp
// state.json — written after every run, read at start of next
var statePath = @"C:\reach\state\outlook-check.json";

// Read previous state
var state = File.Exists(statePath)
    ? JsonSerializer.Deserialize<OutlookCheckState>(File.ReadAllText(statePath))
    : new OutlookCheckState { LastProcessedId = null, LastRunAt = DateTime.MinValue };

// ... do work ...

// Write updated state
state.LastRunAt = DateTime.UtcNow;
state.LastProcessedId = lastEmailId;
File.WriteAllText(statePath, JsonSerializer.Serialize(state, new JsonSerializerOptions { WriteIndented = true }));
```

**What state should track:**
- Last processed item ID (email, record, commit) — prevents reprocessing
- Last run timestamp — detect missed runs or drift
- Run count — useful for detecting loops
- Last known good state — rollback reference if something goes wrong

**SQLite for richer state** — when a flat JSON file isn't enough:

```csharp
#:package Microsoft.Data.Sqlite@8.0.0

// Single file DB, lives alongside the reach scripts
// Tracks processed items, run history, error log
// Queryable — Claude can inspect it during debugging
```

---

## Logging — Structured and Reconstructable

Silent failures in autonomous mode are the hardest to debug. Every reach logs
everything — not to console (no one is watching) but to a structured file.

```csharp
var logPath = @"C:\reach\logs\outlook-check.log";

void Log(string level, string message, object? data = null)
{
    var entry = new
    {
        timestamp = DateTime.UtcNow.ToString("o"),
        level,
        message,
        data
    };
    File.AppendAllText(logPath, JsonSerializer.Serialize(entry) + "\n");
}

Log("INFO", "Run started", new { trigger = "scheduler", version = "1.0" });
Log("INFO", "Outlook connected", new { folderCount = 12 });
Log("INFO", "Emails processed", new { count = 3, skipped = 1 });
Log("ERROR", "Failed to post record", new { requestId = "PROJ-001", error = ex.Message });
Log("INFO", "Run completed", new { duration = elapsed.TotalSeconds });
```

**Log rotation** — append-only logs grow. Rotate weekly:

```csharp
var logFile = $@"C:\reach\logs\{DateTime.Now:yyyy-WW}.log";
```

**What every autonomous run should log:**
- Run start — timestamp, trigger source
- Each source reached — what was found, what was skipped
- Each action taken — what was sent, posted, written
- State transitions — what changed
- Errors — full exception, context, what was being attempted
- Run end — duration, summary counts

---

## Example — Autonomous Outlook Monitor

Wake at 8am, check project emails since last run, categorize, add records, signal if
urgent, sleep.

```csharp
#:package Microsoft.Office.Interop.Outlook@15.0.4797.1004
#:package Microsoft.Data.SqlClient@5.2.0
#:package System.Text.Json@8.0.0

using Outlook = Microsoft.Office.Interop.Outlook;
using Microsoft.Data.SqlClient;
using System.Text.Json;

var statePath = @"C:\reach\state\outlook-monitor.json";
var logPath = $@"C:\reach\logs\outlook-{DateTime.Now:yyyy-MM-dd}.log";
var startedAt = DateTime.UtcNow;

void Log(string level, string msg, object? data = null) =>
    File.AppendAllText(logPath,
        JsonSerializer.Serialize(new { ts = DateTime.UtcNow, level, msg, data }) + "\n");

// Load state
var state = File.Exists(statePath)
    ? JsonSerializer.Deserialize<MonitorState>(File.ReadAllText(statePath))!
    : new MonitorState { LastRunAt = DateTime.UtcNow.AddDays(-1) };

Log("INFO", "Run started", new { lastRun = state.LastRunAt });

try
{
    // Connect to Outlook
    var app = new Outlook.Application();
    var inbox = app.ActiveExplorer().Session.GetDefaultFolder(
        Outlook.OlDefaultFolders.olFolderInbox);

    Log("INFO", "Outlook connected", new { folder = inbox.Name });

    // Filter emails since last run
    var filter = $"[ReceivedTime] >= '{state.LastRunAt:MM/dd/yyyy HH:mm}'";
    var items = inbox.Items.Restrict(filter);
    var projectEmails = items.Cast<Outlook.MailItem>()
        .Where(m => m.Subject?.Contains("PROJ") == true)
        .ToList();

    Log("INFO", "Project emails found", new { count = projectEmails.Count });

    foreach (var email in projectEmails)
    {
        // Categorize and record
        Log("INFO", "Processing email", new { subject = email.Subject, received = email.ReceivedTime });

        // Signal if urgent keywords detected
        if (email.Subject.Contains("URGENT") || email.Subject.Contains("CRITICAL"))
        {
            Log("WARN", "Urgent email detected — signaling", new { subject = email.Subject });
            SignalUrgent(email); // launch notification process, write signal file, etc.
        }
    }

    // Update state
    state.LastRunAt = DateTime.UtcNow;
    state.LastRunCount = projectEmails.Count;
    File.WriteAllText(statePath, JsonSerializer.Serialize(state));

    Log("INFO", "Run completed", new {
        duration = (DateTime.UtcNow - startedAt).TotalSeconds,
        processed = projectEmails.Count
    });
}
catch (Exception ex)
{
    Log("ERROR", "Run failed", new { error = ex.Message, stack = ex.StackTrace });
    // Don't rethrow — let scheduler see clean exit, log has the detail
}

void SignalUrgent(Outlook.MailItem mail)
{
    // Write a signal file another reach watches for
    var signalPath = @"C:\reach\signals\urgent.json";
    File.WriteAllText(signalPath, JsonSerializer.Serialize(new
    {
        subject = mail.Subject,
        from = mail.SenderName,
        received = mail.ReceivedTime,
        signalledAt = DateTime.UtcNow
    }));
}

record MonitorState
{
    public DateTime LastRunAt { get; set; }
    public int LastRunCount { get; set; }
}
```

---

## Multi-Agent Pattern — Signal Files

Reaches communicate through signal files. One reach writes a signal, another
watches for it and acts. Lightweight, no message queue, no infrastructure.

```
reach-A (outlook monitor)
  → detects urgent email
  → writes C:\reach\signals\urgent.json

reach-B (notifier) — triggered by Task Scheduler watching for that file
  → reads urgent.json
  → sends Teams/email notification
  → deletes signal file (consumed)
  → logs action
```

**Signal file convention:**

```
C:\reach\signals\
  urgent.json          # urgent item detected, needs immediate attention
  timesheet-ready.json # draft timesheet ready for review
  db-alert.json        # database anomaly detected
  deploy-complete.json # deployment finished, ready to verify
```

Each signal is a small JSON payload — what happened, when, what data the consumer needs.
The consuming reach deletes the signal after processing — consumed once, gone.

---

## Multi-Agent Pattern — Spawning Child Reaches

One reach spawns another as a child process, waits for output, acts on it.

```csharp
using System.Diagnostics;

var result = await RunReach("query-db.cs", "--env staging --table Equipment");

async Task<string> RunReach(string script, string args = "")
{
    var psi = new ProcessStartInfo("dotnet", $"run {script} {args}")
    {
        WorkingDirectory = @"C:\reach",
        RedirectStandardOutput = true,
        RedirectStandardError = true,
        UseShellExecute = false
    };

    using var proc = Process.Start(psi)!;
    var output = await proc.StandardOutput.ReadToEndAsync();
    var error = await proc.StandardError.ReadToEndAsync();
    await proc.WaitForExitAsync();

    if (proc.ExitCode != 0)
        throw new Exception($"Child reach failed: {error}");

    return output;
}
```

---

## Multi-Agent Pattern — Autonomous Timesheet Draft

Wake nightly, reconstruct the day from git + Outlook + project plan,
draft timesheet entries, write for morning review.

```
11pm — Task Scheduler wakes timesheet-drafter.cs
  ↓
Reads git log for today — what was committed, when
  ↓
Reads Outlook — project emails, meeting invites for today
  ↓
Cross-references project plan — maps work to request IDs
  ↓
Drafts timesheet entries:
  PROJ-001  9:00am-12:00pm   DB investigation + deployment review
  PROJ-001  12:30pm-5:30pm   Deployment fix + stakeholder comms
  ↓
Writes draft to C:\reach\timesheets\2026-06-11-draft.json
  ↓
Writes signal: timesheet-ready.json
  ↓
Morning: you review draft, approve, Playwright posts to your timesheet system
```

---

## Failure Handling Principles

**1. Never let an exception kill the run silently**
Wrap the entire run in try/catch. Log the error in full. Exit cleanly.
Task Scheduler sees a clean exit — the log has the real story.

**2. Idempotent actions where possible**
Design reaches so running them twice doesn't cause double-posting, duplicate records,
or duplicate emails. Check state before acting: *"did I already do this?"*

**3. Signal files are consumed once**
After a consuming reach reads and acts on a signal, it deletes it. Prevents
a second run from re-acting on a stale signal.

**4. Dead letter pattern for failures**
If a reach fails to process an item, move it to a dead letter location rather
than dropping it silently:

```
C:\reach\dead-letter\
  email-12345-failed-2026-06-11.json   # failed item + error context
```
Review dead letters manually. Nothing is lost.

**5. Heartbeat log**
Each scheduled reach writes a heartbeat entry even if there's nothing to do:

```json
{ "ts": "2026-06-11T08:00:01Z", "level": "HEARTBEAT", "msg": "Run completed — nothing to process" }
```
Absence of heartbeat = the reach didn't run. Visible from the log without
having to hunt for missing output.

---

## COM Constraint — App Must Be Running

Outlook COM requires Outlook to be open. For autonomous runs on a locked
but active machine — Outlook running in background is sufficient.

For runs where Outlook may be closed:

```csharp
// Launch Outlook if not running
if (!Process.GetProcessesByName("OUTLOOK").Any())
{
    Process.Start("OUTLOOK");
    await Task.Delay(5000); // wait for COM server to initialize
}
```

For machines where Outlook won't be open (server, overnight) — consider
Microsoft Graph API as the Outlook alternative. Requires auth setup once,
then works headlessly without the app.

---

## Limitations — Honest View

| Limitation | Impact | Mitigation |
|---|---|---|
| No persistent memory | Reaches start fresh each run | Explicit state files or SQLite |
| COM needs app running | Outlook, Excel must be open | Launch on startup or use Graph API |
| Corporate IT policy | Task Scheduler, NuGet may be restricted | Work with IT, use internal NuGet feed |
| UI automation is brittle | Breaks when apps update layout | Use API/DB layer where available |
| Context window in long chains | Quality degrades with accumulated output | Summarize and pass forward, don't append |
| No visual feedback loop | Can't see what's happening unattended | Structured logging, signal files, heartbeat |
| API rate limits | Multi-agent Claude calls can hit limits | Design for low-frequency autonomous runs |

---

## Folder Convention

```
C:\reach\
  scripts\        # .cs reach files
  state\          # JSON state files per reach
  signals\        # inter-reach communication
  logs\           # structured run logs (weekly rotation)
  timesheets\     # drafted timesheet entries pending review
  dead-letter\    # failed items for manual review
  outputs\        # reach outputs for human consumption
```

---

## Notes

- Autonomous REACH runs on the same .NET 10 SDK as interactive REACH
- Task Scheduler runs reaches under your user account — inherits your permissions,
  your Outlook session, your network access
- Reaches exit cleanly after each run — no always-on process, no memory accumulation
- The human stays in the loop for consequential actions (timesheet posting, email sending)
  via the review-and-approve pattern — draft autonomously, act with confirmation
- Start with read-only autonomous reaches before adding write actions —
  build trust in the pattern before it acts on your behalf
