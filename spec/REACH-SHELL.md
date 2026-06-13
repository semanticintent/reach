# REACH Shell — Spectre.Console TUI Sketch

REACH Shell is a lightweight terminal UI wrapping the REACH workflow.
Built with Spectre.Console. Runs anywhere .NET 10 runs.
No install. No account. No cloud. No Electron.

Philosophy: the terminal is the soul. The shell is the cormorant.

---

## The Cormorant Structure

REACH Shell maps directly to the three cormorant foraging dimensions:

```
PERCH  →  Space    Saved reaches. Elevated positions you return to.
DIVE   →  Sound    The terminal. Where you go in. Signal and urgency.
WAKE   →  Time     History. What persists after you surface.
```

Every REACH session is a foraging dive:
- You perch — select a saved reach
- You dive — execute it against live sources
- You surface — findings return
- The wake persists — artifact written to history

---

## Layout

```
┌─────────────────────────────────────────────────────────────────┐
│  REACH Shell                                          v1.0  [?] │
├─────────────────┬───────────────────────────┬───────────────────┤
│                 │                           │                   │
│  PERCH          │  DIVE                     │  WAKE             │
│  ─────────────  │  ──────────────────────   │  ───────────────  │
│                 │                           │                   │
│  Saved Reaches  │  > reach outlook.inbox    │  Today            │
│                 │    where subject          │  ▸ proj-90-rpm     │
│  ▸ proj-90-rpm   │      contains "[PROJECT]"       │    🔴 all-red     │
│    sprint-rev   │    since 90-days-ago       │    28% after-hrs  │
│    db-staging   │    analyze rpm            │                   │
│    deploy-chk   │    output report          │  Yesterday        │
│    timesheet    │                           │  ▸ sprint-review  │
│                 │  ──────────────────────   │    🟡 yellow      │
│  [N] New        │                           │    22% after-hrs  │
│  [D] Delete     │  Running... ⠋             │                   │
│  [E] Edit       │                           │  This Week        │
│                 │  RPM: 🔴 High             │  ▸ db-staging     │
│                 │  Emails: 47 matched       │  ▸ deploy-chk     │
│                 │  After-hours: 28%         │                   │
│                 │  vs target: 15%           │  [↑↓] Navigate    │
│                 │                           │  [Enter] Open     │
│                 │  Recommendation:          │  [D] Diff         │
│                 │  Open valley week         │                   │
│                 │                           │                   │
├─────────────────┴───────────────────────────┴───────────────────┤
│  [Tab] Switch panel  [R] Run  [S] Save  [Q] Quit  [/] Search   │
└─────────────────────────────────────────────────────────────────┘
```

Three panels. One status bar. Keyboard driven. Nothing else.

---

## Panel Behaviour

### PERCH — Left Panel (1/4 width)

Saved `.reach` files from `C:\reach\saved\`.
Loaded on startup. Refreshed on file system change.

```
Navigation:  ↑↓ arrows to move selection
             Enter to load into DIVE panel
             N to create new reach (opens editor)
             E to edit selected reach
             D to delete selected reach
             / to search saved reaches
```

Each item shows:
- Reach name
- Last run timestamp
- Last run RPM state (if sprint-type reach)

```
▸ proj-90-rpm          ← currently selected
    last: today 09:14
    🔴 all-red

  sprint-review
    last: yesterday
    🟡 yellow

  db-staging
    last: 3 days ago
```

---

### DIVE — Center Panel (2/4 width)

The terminal. Input at the top. Output streams below.
This is where the reach runs. Everything else is chrome around this.

```
Input area:   editable reach block
              syntax-highlighted via Spectre markup
              Tab to autocomplete source names
              Enter to run

Output area:  streams as reach executes
              spinner while running
              RPM verdict with color on completion
              prompt gate surfaces here if write action pending
```

**Input — syntax highlighting:**

```
[cyan]reach[/] [green]outlook.inbox[/] [grey]+[/] [green]git.commits[/]
  [dim]where[/]     [white]subject contains "[PROJECT]"[/]
  [dim]since[/]     [white]90-days-ago[/]
  [dim]analyze[/]   [yellow]rpm[/]
  [dim]benchmark[/] [white]after-hours 15%[/]
  [dim]output[/]    [white]report[/]
```

**Output — streaming:**

```
⠋ Connecting to Outlook...
✓ Outlook connected — inbox loaded
⠋ Reading project emails since 90 days ago...
✓ 47 emails matched
⠋ Reading git commits...
✓ 127 commits — analyzing time distribution
⠋ Calculating RPM...

────────────────────────────────────
RPM STATE        🔴 All Red
After-hours      28%  (target: 15%)
Email volume     47 matched / 602 total
Meeting density  High — 3 dense weeks
Commits          127 total, 36 after-hours

RECOMMENDATION
Open valley week — 1 week deload
Reframe: strong run, well-managed
────────────────────────────────────

Artifact written → history/2026-06-13-proj-90-rpm.reach-artifact
```

**Prompt gate — inline confirm:**

When a write action is pending, the prompt surfaces in the DIVE panel:

```
────────────────────────────────────
  Draft email ready for review

  To:       team@company.com
  Subject:  [PROJECT] Update — June 13
  Body:     3 paragraphs from analysis

  [S] Send    [D] Save Draft    [X] Discard
────────────────────────────────────
```

---

### WAKE — Right Panel (1/4 width)

History of all previous runs. Reads from `C:\reach\history\`.
Grouped by time — today, yesterday, this week, older.

```
Navigation:  ↑↓ to move selection
             Enter to open artifact in DIVE panel (read-only)
             D to diff two selected artifacts
             X to delete
```

Each entry shows:
- Reach name
- RPM state emoji if applicable
- Key metric from the artifact

```
Today
▸ proj-90-rpm
    🔴 28% after-hrs

Yesterday
  sprint-review
    🟡 22% after-hrs

  db-staging
    47 rows returned

This Week
  deploy-chk
    ✓ all green

  timesheet
    ✓ posted 2 entries
```

**Artifact view** — when you open a history entry, DIVE panel shows
the `.reach-artifact` in read-only mode with full findings.

**Diff view** — select two history entries, press D:

```
sprint-review — June 6 vs June 13
────────────────────────────────────
RPM          🟡 yellow    →  🔴 all-red   ↑ worse
After-hours  22%          →  28%          ↑ worse
Meetings     moderate     →  high density ↑ worse
Commits      steady       →  compressed   ↑ worse

Trend: deteriorating. Third week of upward RPM.
Recommendation: structural intervention needed.
────────────────────────────────────
```

---

## Status Bar

Always visible. One line. Keyboard shortcuts only.

```
[Tab] Panel  [R] Run  [S] Save reach  [H] History  [Q] Quit  [/] Search  [?] Help
```

---

## Keyboard Map

| Key | Action |
|---|---|
| `Tab` | Cycle between PERCH / DIVE / WAKE panels |
| `↑↓` | Navigate list in PERCH or WAKE |
| `Enter` | Load selected reach / open artifact |
| `R` | Run current reach in DIVE |
| `S` | Save current DIVE content as named reach |
| `N` | New reach in DIVE |
| `E` | Edit selected PERCH reach |
| `D` | Delete (PERCH) or Diff two selected (WAKE) |
| `/` | Search saved reaches or history |
| `Esc` | Cancel / clear input |
| `Q` | Quit |
| `?` | Help overlay |

---

## File Structure

```
C:\reach\
  reach.exe           ← single executable, double-click or terminal
  saved\
    proj-90-rpm.reach
    sprint-review.reach
    db-staging.reach
    deploy-check.reach
    timesheet-daily.reach
  history\
    2026-06-13-proj-90-rpm.reach-artifact
    2026-06-12-sprint-review.reach-artifact
    2026-06-11-db-staging.reach-artifact
  state\
    shell.json        ← last selected reach, panel focus, window size
  logs\
    shell.log         ← shell errors only, not reach output
```

---

## Implementation Sketch

Single C# file. Claude generates it. `dotnet run reach-shell.cs` or compiled to `reach.exe`.

```csharp
#:package Spectre.Console@0.49.0
#:package System.Text.Json@8.0.0

using Spectre.Console;
using Spectre.Console.Rendering;

// Three-column layout — PERCH / DIVE / WAKE
var layout = new Layout("Root")
    .SplitColumns(
        new Layout("Perch").Ratio(1),
        new Layout("Dive").Ratio(2),
        new Layout("Wake").Ratio(1));

// Load saved reaches
var savedReaches = LoadSavedReaches(@"C:\reach\saved");
var history = LoadHistory(@"C:\reach\history");
var currentReach = savedReaches.FirstOrDefault();

// PERCH panel — saved reaches list
layout["Perch"].Update(
    new Panel(RenderPerch(savedReaches, currentReach))
        .Header("[cyan]PERCH[/]")
        .BorderColor(Color.Grey)
        .Expand());

// DIVE panel — terminal input + output
layout["Dive"].Update(
    new Panel(RenderDive(currentReach))
        .Header("[yellow]DIVE[/]")
        .BorderColor(Color.Yellow)
        .Expand());

// WAKE panel — history
layout["Wake"].Update(
    new Panel(RenderWake(history))
        .Header("[blue]WAKE[/]")
        .BorderColor(Color.Grey)
        .Expand());

// Live update loop
await AnsiConsole.Live(layout)
    .AutoClear(false)
    .Overflow(VerticalOverflow.Ellipsis)
    .StartAsync(async ctx =>
    {
        while (true)
        {
            var key = Console.ReadKey(intercept: true);
            HandleKeypress(key, layout, ctx);
            ctx.Refresh();
        }
    });
```

**Panel renders** use Spectre markup throughout:

```csharp
IRenderable RenderPerch(List<ReachFile> reaches, ReachFile? selected)
{
    var table = new Table().NoBorder().HideHeaders();
    table.AddColumn("");

    foreach (var r in reaches)
    {
        var isSelected = r == selected;
        var prefix = isSelected ? "[yellow]▸[/] " : "  ";
        var rpm = r.LastArtifact?.Rpm switch
        {
            "all-red" => "[red]🔴[/]",
            "red"     => "[red]🔴[/]",
            "yellow"  => "[yellow]🟡[/]",
            "green"   => "[green]🟢[/]",
            _         => "   "
        };
        table.AddRow($"{prefix}[white]{r.Name}[/] {rpm}");
        if (r.LastArtifact != null)
            table.AddRow($"   [dim]{r.LastArtifact.Date:MMM dd HH:mm}[/]");
    }

    return table;
}
```

---

## Spectre Primitives Used

| Need | Spectre primitive |
|---|---|
| Three-panel layout | `Layout` with `SplitColumns` |
| Named panels | `layout["Perch"]`, `layout["Dive"]`, `layout["Wake"]` |
| Live refresh | `AnsiConsole.Live()` |
| Colored text | Markup — `[red]`, `[cyan]`, `[dim]`, `[yellow]` |
| Bordered panels | `Panel` with `Header` and `BorderColor` |
| Selection list | `SelectionPrompt` for search overlay |
| Progress/spinner | `AnsiConsole.Status()` during reach execution |
| Tables | `Table` with `NoBorder()` for lists |
| Rule/divider | `Rule` for output section separators |
| Keyboard input | `Console.ReadKey` + key dispatch |

---

## Cormorant Identity

The shell is named and structured after the cormorant foraging pattern.
The three panels are not arbitrary — they map to the Sound/Space/Time framework:

```
PERCH  →  Space     Elevated. Observing. Returning to known positions.
DIVE   →  Sound     Signal and urgency. The active dive. Output as chirp.
WAKE   →  Time      Memory. Persistence. The trail that outlasts the session.
```

The cormorant doesn't dive randomly.
It perches, observes, selects, dives with precision,
surfaces with something, returns.

Every REACH session follows that arc.
The shell makes it visible.

---

## Design Constraints

```
✓  Single executable — reach.exe or dotnet run reach-shell.cs
✓  No install beyond .NET 10 SDK
✓  No account, no cloud, no sync
✓  No database — flat files only
✓  Terminal is primary — UI is chrome, not the product
✓  Keyboard driven — mouse optional
✓  Cross-platform — Windows, macOS, Linux
✗  No Electron
✗  No web server
✗  No always-on process
✗  No settings wizard
```

---

## What This Is Not

The shell does not replace Claude Code CLI. It is not an agent runner.
It does not author `.reach` files — Claude does that in a conversation.

The shell is the **surface between you and your saved reaches**.
A memory panel for what you've run. A launch pad for what you're about to run.
A wake trail for what mattered.

Claude authors. REACH executes. The shell remembers.

---

## Notes

- Spectre.Console `Layout` with live update is the right primitive for three-panel TUI
- Known limitation: Spectre `Layout` doesn't support interactive input *inside* a panel natively — the DIVE input area uses a separate `ReadKey` loop outside the layout, then updates the panel content. This is the standard Spectre pattern for interactive TUIs.
- For richer input inside panels — `spectre.tui` (in development by the Spectre team) or `RazorConsole` wraps Spectre with Razor components and full focus management
- Start with the three-panel static layout first, add live update second, add keyboard navigation third — build trust in each layer before adding the next
