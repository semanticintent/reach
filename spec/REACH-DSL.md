# REACH DSL — Vocabulary Specification

REACH DSL is a declarative intent layer that sits above C# and NuGet.
You write what you want. Claude compiles it to the right reach behind the scenes.
No NuGet packages visible. No COM Interop. No Playwright instantiation. Just intent.

Extension: `.reach`
Execution: Claude reads the file, generates typed C# per source block, runs it, returns results.

---

## Syntax Shape

```
reach <source>
  <qualifier>  <value>
  <qualifier>  <value>
  <action>     <value>
  output       <format>
```

Multiple sources compose with `+`:

```
reach <source-a> + <source-b> + <source-c>
  since     <timeframe>
  analyze   <intent>
  output    <format>
```

Everything is lowercase. Indentation is two spaces. Quoted strings for values with spaces.
Unquoted for keywords and identifiers.

---

## Source Vocabulary

### `outlook.inbox`
Reads emails from Outlook inbox. Requires Outlook running.

```
reach outlook.inbox
  where     subject contains "[PROJECT]"
  since     start-of-year
  group by  month
  output    cadence
```

### `outlook.folder`
Reads any named Outlook folder.

```
reach outlook.folder "Deleted Items"
  count     all
  output    summary
```

### `outlook.calendar`
Reads calendar entries — meetings, appointments, events.

```
reach outlook.calendar
  since     10-weeks-ago
  analyze   density
  group by  week
  output    summary
```

### `outlook.sent`
Reads sent items — useful for communication cadence from your side.

```
reach outlook.sent
  where     subject contains "[PROJECT]"
  since     start-of-year
  group by  month
  output    cadence
```

---

### `git.commits`
Reads git commit history. Runs in current directory or specified path.

```
reach git.commits
  since     10-weeks-ago
  analyze   cadence
  output    summary
```

```
reach git.commits
  since     10-weeks-ago
  analyze   after-hours
  benchmark 15%
  output    report
```

### `git.log`
Raw git log with custom shape — when you need more than commits.

```
reach git.log
  since     start-of-month
  format    author + date + message
  output    table
```

---

### `screen`
Captures full screen. Returns image path. Claude reads it inline.

```
reach screen
  save as   snapshot
```

### `screen.window`
Captures a specific named window.

```
reach screen.window "Chrome"
  save as   snapshot
```

### `screen.region`
Captures a defined pixel region.

```
reach screen.region 0 0 1920 1080
  save as   snapshot
```

---

### `browser.page`
Opens a URL in headless Chromium. Extracts content or screenshots.

```
reach browser.page "https://staging.internal/deployments"
  wait for  ".deployment-status"
  extract   text
  output    summary
```

```
reach browser.page "https://staging.internal/dashboard"
  wait for  ".data-table"
  screenshot element ".data-table"
  output    snapshot
```

### `browser.auth`
Navigates through a login flow before reaching the target page.

```
reach browser.auth "https://internal-tool/login"
  field     "#username" "myuser"
  field     "#password" "mypass"
  submit    "button[type=submit]"
  then      browser.page "https://internal-tool/dashboard"
  extract   text
  output    summary
```

---

### `db.query`
Runs a SQL query against a named environment.

```
reach db.query
  env       staging
  sql       "SELECT TOP 100 * FROM Equipment WHERE Status = 'Active'"
  output    json
```

### `db.table`
Inspects a table — schema, row count, sample data.

```
reach db.table Equipment
  env       staging
  sample    20
  output    summary
```

---

### `file.read`
Reads one or more files. Supports `.txt`, `.docx`, `.pdf`.

```
reach file.read "attachments/report.docx"
  extract   text
  output    plain
```

```
reach file.read "attachments/*.pdf"
  extract   text
  merge     true
  output    plain
```

### `file.merge`
Merges multiple text files into one.

```
reach file.merge "attachments/*.txt"
  separator "---"
  output    "merged_context.txt"
```

### `file.watch`
Watches a folder for new files and signals when they arrive.

```
reach file.watch "C:\reach\attachments"
  on new    signal "file-arrived"
  output    log
```

---

### `process.launch`
Launches an application.

```
reach process.launch "OUTLOOK"
  wait for  ready
```

### `process.signal`
Writes a signal file for inter-reach communication.

```
reach process.signal "urgent"
  payload   subject from "email"
  output    log
```

---

### `ui.window`
Interacts with a desktop application's UI control tree via UIAutomation.

```
reach ui.window "OUTLOOK"
  find      name "Inbox"
  read      items
  output    list
```

```
reach ui.window "OUTLOOK"
  find      name "Filter"
  click
  output    log
```

---

### `http.get`
Calls a REST endpoint.

```
reach http.get "https://internal-api/deployment/status"
  output    json
```

### `http.post`
Posts to a REST endpoint.

```
reach http.post "https://internal-api/notify"
  body      json from "signal"
  output    log
```

---

## Qualifier Vocabulary

| Qualifier | Applies to | Meaning |
|---|---|---|
| `where` | outlook.*, db.* | Filter condition |
| `since` | outlook.*, git.*, calendar | Time range start |
| `until` | outlook.*, git.*, calendar | Time range end |
| `group by` | any | Group results by dimension |
| `analyze` | any | Named analysis intent |
| `benchmark` | git.commits, any metric | Target value for comparison |
| `wait for` | browser.* | CSS selector to wait for before acting |
| `extract` | browser.*, file.* | What to pull — text, table, links |
| `screenshot` | browser.*, screen.* | Capture visual output |
| `find` | ui.* | Control to locate by name or type |
| `click` | ui.* | Click the found control |
| `field` | browser.auth | Fill a form field |
| `submit` | browser.auth | Submit a form |
| `sample` | db.* | Number of rows to sample |
| `merge` | file.* | Combine multiple file outputs |
| `separator` | file.merge | Delimiter between merged files |
| `save as` | screen.* | Output filename |
| `payload` | process.signal | Data to include in signal |
| `on new` | file.watch | Event trigger |
| `env` | db.* | Named environment (dev/staging/prod) |
| `format` | git.log | Output shape |
| `count` | any | Count items instead of listing them |

---

## Output Vocabulary

| Output | Meaning |
|---|---|
| `summary` | Claude narrates what it found |
| `cadence` | Frequency pattern over time — counts by period |
| `report` | Structured findings with interpretation |
| `json` | Raw JSON to stdout |
| `table` | Formatted table |
| `list` | Line-by-line items |
| `plain` | Plain text extraction |
| `snapshot` | Image file — Claude reads inline |
| `log` | Append to run log |
| `signal` | Write signal file for another reach |

---

## Analyze Vocabulary

Named analysis intents Claude understands:

| Analyze value | What Claude does |
|---|---|
| `cadence` | Groups by time period, identifies patterns, gaps, spikes |
| `after-hours` | Splits activity by time of day, calculates % outside core hours |
| `density` | Measures concentration — meetings per week, emails per day |
| `rpm` | Reads combined signal across sources, calls green/yellow/red/all-red |
| `velocity` | Rate of output over time — commits, emails, deployments |
| `sentiment` | Tone pattern across communications |
| `recovery` | What it would take to normalize current state |
| `summary` | Plain language synthesis of what the data shows |

---

## Timeframe Vocabulary

| Value | Meaning |
|---|---|
| `today` | Current day |
| `this-week` | Monday to now |
| `last-week` | Previous full week |
| `start-of-month` | First day of current month |
| `start-of-year` | January 1 |
| `10-weeks-ago` | N weeks before today |
| `30-days-ago` | N days before today |
| `"2026-01-01"` | Explicit date |

---

## Composition — Multiple Sources

Sources compose with `+`. Claude reaches into each, holds all results in context,
synthesizes across them.

```
reach outlook.inbox + outlook.calendar + git.commits
  since     10-weeks-ago
  analyze   rpm
  benchmark after-hours 15%
  output    report
```

This is the sprint cadence review in one block. Claude reaches into all three sources,
reads the combined signal, calls the RPM state, flags the after-hours against the benchmark,
and delivers a narrative report.

---

## Named Reaches — Reusable Patterns

A `.reach` file can define a named pattern for reuse:

```
# sprint-review.reach

name        sprint-cadence-review
description Read sprint health across email, calendar, and commits

reach outlook.inbox + outlook.calendar + git.commits
  since     10-weeks-ago
  analyze   rpm
  benchmark after-hours 15%
  output    report
```

Run it:
```
dotnet reach sprint-review.reach
```

Or in Claude CLI:
```
run sprint-review.reach
```

Share it. Fork it. The pattern travels as a file.

---

## Artifact Output — `.reach-artifact`

Every reach that produces findings writes a `.reach-artifact` — the typed,
human-readable, git-diffable record of what was found.

```
REACH      sprint-cadence-review
ID         proj.2026.sprint-10
DATE       2026-06-11
─────────────────────────────────────────
period:    2026-04-01 to 2026-06-11
rpm:       all-red
after-hours: 30%
target:    10-15%

sources:
  outlook.inbox:    602 emails matched
  outlook.calendar: 47 meetings
  git.commits:      127 total, 38 after-hours

finding: |
  Sustained redline across all 10 weeks.
  No week dropped below high RPM.
  30% after-hours structural, not occasional.

recommendation:
  recovery: open-valley-week
  duration: 1 week
  reframe:  strong-run-well-managed
```

Plain text. No server. No database. Lives in your project. Git tracked.
Claude reads it next sprint without re-running everything.

---

## Interaction Layer — Pause and Ask

REACH has three modes:

```
Read   →  reach into sources, return findings
Act    →  reach into systems, change state
Pause  →  surface a prompt, wait for human, continue
```

The **Pause** mode keeps the human in the loop at the right moment — not just at the
end of a flow but *inside* it, before consequential actions. REACH acts autonomously
on reads. REACH pauses on writes.

The prompt layer is the conscience of the automation.

---

### `prompt.confirm`
Simplest gate. Yes/No before proceeding. Renders a native Windows dialog.

```
reach outlook.draft
  to        "team@company.com"
  subject   "[PROJECT] Deployment Update"
  body      from git.commits since today
  then      prompt.confirm
    title   "Send this email?"
    on yes  outlook.send
    on no   outlook.save-draft
```

---

### `prompt.preview`
Shows the drafted artifact — email, report, query result — before acting.
Renders a preview panel with named action buttons.

```
reach db.query
  env     staging
  sql     "SELECT * FROM Equipment WHERE Status = 'Pending'"
  then    prompt.preview
    title   "Review before posting to your timesheet system"
    actions approve | edit | cancel
    on approve  timesystem.post
    on edit     return to input
    on cancel   stop
```

```
reach outlook.draft
  to      "team@company.com"
  subject "Daily Summary — {date}"
  body    from git.commits + outlook.inbox since today analyze summary
  then    prompt.preview
    title   "Email draft ready"
    actions send | edit | save-draft | discard
    on send       outlook.send
    on save-draft outlook.save
    on edit       return to draft
    on discard    stop
```

---

### `prompt.input`
Asks for a single value mid-flow. Renders a minimal Windows input form.

```
reach outlook.draft
  to      from prompt.input
    label "Who should receive this?"
    type  email
  subject from prompt.input
    label "Subject"
    default "[PROJECT] Update — {date}"
  body    from git.commits since today
  then    prompt.preview
    actions send | save-draft
    on send       outlook.send
    on save-draft outlook.save
```

---

### `prompt.form`
Multi-field form before REACH proceeds. Fields can be text, time, datetime,
select, textarea, email-list, checkbox. Renders as a WinForms dialog.

```
reach timesystem.timesheet
  then  prompt.form
    title  "Review timesheet entries"
    fields
      request-id   text      "Request ID"
      start-time   time      "Start"
      end-time     time      "End"
      description  textarea  "Description"
      environment  select    "dev | staging | prod"
    on submit  timesystem.post
    on cancel  outlook.save-draft
```

---

### Prompt UI Toolkit

Claude chooses the right Windows UI primitive per prompt type:

| Prompt type | Claude reaches for | Feel |
|---|---|---|
| `prompt.confirm` | `System.Windows.Forms.MessageBox` | Native Windows dialog |
| `prompt.input` | WinForms single-field form | Lightweight, instant |
| `prompt.form` | WinForms multi-field dialog | Full form, your fields |
| `prompt.preview` | WinForms panel with action buttons | Preview + decision |
| Rich preview | Local HTML via Playwright | Browser-rendered, flexible |

All generated on demand. No pre-built UI components. The form is authored for the
specific task — just like the reach itself.

---

## New Sources — Write and CI/CD

### `outlook.draft`
Composes an email in Outlook. Does not send until explicitly instructed.

```
reach outlook.draft
  to        "stakeholder@company.com"
  subject   "Sprint Summary — {date}"
  body      from analyze.summary
  then      prompt.preview
    actions send | save-draft | discard
    on send       outlook.send
    on save-draft outlook.save
```

### `outlook.send`
Sends the composed draft. Always preceded by `prompt.confirm` or `prompt.preview`
unless explicitly marked `no-prompt` — a deliberate friction point.

### `outlook.meeting`
Composes a meeting invite with attendees, time, and agenda.

```
reach outlook.meeting
  title     from prompt.input
    label   "Meeting title"
  attendees from prompt.input
    label   "Attendees"
    type    email-list
  date      from prompt.input
    label   "Date and time"
    type    datetime
  duration  from prompt.input
    label   "Duration (minutes)"
    default 30
  body      from prompt.input
    label   "Agenda"
    type    textarea
  then      prompt.preview
    actions send | save-draft
    on send       outlook.meeting.send
    on save-draft outlook.meeting.save
```

### `ci.pipeline`
Watches a CI/CD pipeline URL for status. Triggers on state change.

```
reach ci.pipeline
  watch     "https://internal-ci/build/latest"
  on fail   prompt.confirm
    title   "Build failed — notify team?"
    detail  from ci.error-log
    on yes  outlook.draft
              to      "team@company.com"
              subject "Build failure — {branch}"
              body    from ci.error-log
            then prompt.preview
              on send outlook.send
    on no   process.signal "build-failed"
```

### `ci.error-log`
Reads the error output from the most recent CI pipeline run.

```
reach ci.error-log
  env     staging
  output  summary
```

### `timesystem.timesheet`
Posts timesheet entries to your timesheet system. Requires prompt gate before posting.

```
reach timesystem.timesheet
  entry
    request-id  "PROJ-001"
    start       "09:00"
    end         "12:00"
    description "DB investigation"
  entry
    request-id  "PROJ-001"
    start       "12:30"
    end         "17:30"
    description "Deployment fix + comms"
  then  prompt.preview
    title   "Timesheet entries — review before posting"
    actions post | edit | discard
    on post timesystem.post
```

---

## Interaction Qualifier Vocabulary

| Qualifier | Meaning |
|---|---|
| `then` | Chain next action after current completes |
| `on <outcome>` | Branch based on prompt result |
| `actions` | Define available buttons in a preview prompt |
| `return to` | Loop back to a previous step |
| `default` | Pre-fill a field value |
| `detail` | Additional context shown in the prompt |
| `label` | Field label in a form or input prompt |
| `type` | Field type: text, email, time, datetime, textarea, select, email-list, checkbox |
| `fields` | Opens a multi-field block inside prompt.form |
| `no-prompt` | Explicit override — skip the pause gate (use with care) |

---

## Full Flow Example — Daily Timesheet from Git

Autonomous draft at end of day, human reviews and posts.

```
# timesheet-daily.reach

name        daily-timesheet-draft
description Reconstruct today's work from git and calendar, draft timesheet entries

reach git.commits + outlook.calendar + outlook.inbox
  since     today
  analyze   summary
  then timesystem.timesheet
    draft from analyze.summary
    map   commits to request-ids via project-plan
  then prompt.form
    title  "Review today's timesheet"
    fields
      entry-1-request   text  "Morning request ID"
      entry-1-desc      textarea "Morning description"
      entry-2-request   text  "Afternoon request ID"
      entry-2-desc      textarea "Afternoon description"
    on submit timesystem.post
    on cancel outlook.save-draft
```

Wake at 5pm. Claude reconstructs the day. Form surfaces with pre-filled entries.
You adjust if needed. Submit. Posted. Done.

---

## REACH Modes Summary

```
READ    reach outlook.inbox where subject contains "[PROJECT]"
        → finds, groups, summarizes. No side effects.

ACT     reach process.launch "OUTLOOK"
        → changes machine state. Low risk, no data written.

PAUSE   reach outlook.draft ... then prompt.preview
        → surfaces UI, waits for human, proceeds on approval.
        → default mode for all write operations.
```

The design principle:
- **Reads** run freely — no prompt needed
- **Writes** pause by default — prompt gate required
- **`no-prompt`** is an explicit override — a deliberate choice, not a default

---

## The Compilation Model

```
developer writes      →  sprint-review.reach
Claude reads DSL      →  understands intent per source block
Claude generates      →  typed C# per source (NuGet chosen per block)
dotnet run            →  executes each reach
prompt surfaces       →  if write action — Windows UI appears, human decides
results flow back     →  Claude synthesizes across all sources
artifact written      →  sprint-review.reach-artifact
```

The developer never sees:
- NuGet package names
- COM Interop calls
- Playwright instantiation
- SqlClient connection strings (beyond env name)
- WinForms boilerplate for prompt UI

Just intent. Just results. Just the right pause at the right moment.

---

## Design Principles

**1. Reads like intent, not code**
Anyone can read a `.reach` file and understand what it does. No training required.

**2. Sources are the vocabulary, not the implementation**
`outlook.inbox` is a concept. How Claude reaches it — COM Interop, Graph API, whatever
is appropriate — is Claude's decision, not yours.

**3. Composition is first-class**
`+` between sources is the most powerful operator in the language. The sprint review,
the timesheet draft, the deployment check — all are compositions of simple sources.

**4. Artifacts outlast sessions**
Every meaningful reach produces a `.reach-artifact`. The finding is permanent even
though the script is ephemeral. Memory without infrastructure.

**5. Benchmarks make findings actionable**
`benchmark after-hours 15%` turns a number into a verdict. Claude doesn't just report
30% — it calls it against the target and recommends what normalizes it.

**6. Named reaches are the sharing unit**
You share a `.reach` file. Someone forks it, changes the project name, runs it.
That's the adoption flywheel. Like jQuery plugins — small, composable, shareable.

**7. Writes pause by default**
Reads run freely. Writes surface a prompt gate. The human approves before anything
is sent, posted, or created. `no-prompt` is an explicit override — never a default.
The prompt layer is the conscience of the automation.

---

## What This Is Not

- Not a programming language — nothing in a `.reach` file executes directly
- Not a configuration file — it expresses intent, not settings
- Not a query language — it orchestrates multiple sources, not one
- Not a framework — no install, no setup, no infrastructure
- Not CAL — CAL computes cascade analysis. REACH reaches into live systems.

Both are methodology-as-infrastructure. Different domains. Different jobs.

---

## Relationship to the Semantic Intent Ecosystem

| DSL | Domain | Execution model |
|---|---|---|
| CAL | Cascade analysis | Deterministic executor → scores and alerts |
| EMBER | Legacy modernization | Typed artifact carrier → agent handoffs |
| REACH | Windows workflow automation | Intent compiler → C# reaches → live results |

Each instantiates Methodology-as-Infrastructure at a different layer.
Each has a small vocabulary. Each produces typed artifacts.
Each is human-readable without a manual.

---

## Citation

```
@misc{shatny2026reach,
  author    = {Shatny, Michael},
  title     = {REACH: Runtime Executable Adaptive C# Handler — DSL Specification},
  year      = {2026},
  publisher = {npm},
  package   = {@semanticintent/reach},
  url       = {https://reach.semanticintent.dev}
}
```
