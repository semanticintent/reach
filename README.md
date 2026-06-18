# REACH + OCTO

[![DOI](https://zenodo.org/badge/DOI/10.5281/zenodo.20680385.svg)](https://doi.org/10.5281/zenodo.20680385)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

**REACH** — a local workflow practice where Claude authors single-file C# scripts on demand, reaching into live systems (Outlook, SQL Server, git, browsers, desktop apps) to find, analyze, draft, and act.

**OCTO** — an orchestration layer built on REACH. Multiple arms reach simultaneously, findings combine, a contextual decision surface appears, the human decides, the session closes with personality.

Neither requires infrastructure. Neither is always-on. Both treat human judgment as a first-class architectural primitive — not a safety check bolted on reluctantly.

---

## The One-Line Version

**REACH:** Declare intent in a `.reach` file. Claude compiles it to typed C# and runs it against your live systems.

**OCTO:** Declare multiple arms and a decision surface in a `.octo` file. Claude runs them, reads the combined signal, and surfaces the right interface for this moment.

---

## How a REACH Works

```
dotnet run task.cs
```

Each reach declares its own dependencies inline — no `.csproj`, no scaffolding:

```csharp
#:package Microsoft.Office.Interop.Outlook@15.0.4797.1004

using Outlook = Microsoft.Office.Interop.Outlook;

var app = new Outlook.Application();
var inbox = app.ActiveExplorer().Session.GetDefaultFolder(
    Outlook.OlDefaultFolders.olFolderInbox);
// ...
```

Claude writes the file, runs it, reads stdout/stderr, fixes compile errors, reruns. The capability surface is unlimited — Claude chooses the library, not you.

---

## The REACH DSL

A `.reach` file expresses intent:

```
reach outlook.inbox + git.commits
  since     10-weeks-ago
  analyze   rpm
  benchmark after-hours 15%
  output    report
```

Claude reads this, generates the right C# per source block, runs it, synthesizes across all sources, and writes a `.reach-artifact` — the typed, human-readable, git-diffable record of what was found.

See [REACH-DSL.md](spec/REACH-DSL.md) for the full vocabulary.

---

## How an OCTO Works

A `.octo` file orchestrates multiple arms:

```
# morning.octo

name    morning-brief
arms
  - outlook.inbox     since yesterday     analyze urgent
  - teams.planner     status overdue
  - teams.transcript  latest              summarize
  - git.commits       since yesterday     analyze cadence

surface
  title   "Good morning. Here's what needs you."
  show    urgent-items
  actions
    draft-reply   →  reach outlook.draft from selected
    send-summary  →  reach outlook.draft to team summarize all
    snooze        →  reach teams.planner snooze selected

close
  tone  warm
  joke  true
```

Arms run simultaneously. OCTO reads the combined signal. A contextual surface appears — not pre-built, authored for this moment. Human decides. Session closes with personality.

See [OCTO.md](spec/OCTO.md) for the full pattern.

---

## The Three Modes

```
READ   →  reach into sources, return findings — no side effects, runs freely
ACT    →  change state — always gated by surface confirmation
PAUSE  →  surface a prompt, wait for human, continue
```

Reads run freely. Writes pause by default. The surface is the conscience of the automation.

---

## Why Not a Framework

| | Enterprise agentic frameworks | REACH + OCTO |
|---|---|---|
| Setup | Infrastructure required | .NET 10 SDK |
| Tokens | Continuous burn | Per task — bounded |
| Human | Routed around | First-class architectural primitive |
| Output | Whenever agents decide | When you ask |
| State | Framework-managed, complex | Flat files — owned by you |

REACH + OCTO are not competitors to LangChain, AutoGen, or CrewAI. They answer a different question:

> *"How does one person work better tomorrow morning?"*

See [OCTO-vs-AGENTIC.md](spec/OCTO-vs-AGENTIC.md) for the full positioning.

---

## Activating REACH in Claude Code

Clone this repo. Open `CLAUDE.md` in your Claude Code session. The runtime context loads — Claude now knows the REACH vocabulary, the compilation model, the reach index, and the interaction layer. You're running REACH.

```bash
git clone https://github.com/semanticintent/reach
cd reach
# Open CLAUDE.md in Claude Code to activate the practice
```

---

## Spec Files

| File | Contents |
|---|---|
| [REACH-DSL.md](spec/REACH-DSL.md) | Source, qualifier, output, analyze, and timeframe vocabulary |
| [REACH-AUTONOMOUS.md](spec/REACH-AUTONOMOUS.md) | Autonomous + multi-agent patterns, state management, failure handling |
| [REACH-SHELL.md](spec/REACH-SHELL.md) | Spectre.Console TUI — PERCH / DIVE / WAKE |
| [OCTO.md](spec/OCTO.md) | Orchestration pattern, `.octo` format, surface types, workflow examples |
| [OCTO-HEALTH.md](spec/OCTO-HEALTH.md) | Tray icon, health panel, daily report, weekly summary email |
| [OCTO-vs-AGENTIC.md](spec/OCTO-vs-AGENTIC.md) | Positioning vs enterprise agentic frameworks |

---

## Examples

See [examples/](examples/) for ready-to-run `.reach` and `.octo` files.

---

## In the Semantic Intent Ecosystem

```
CAL     →  decides          cascade analysis, scoring
EMBER   →  remembers        typed SIL artifacts, agent handoffs
REACH   →  reaches          single arm, one source, live systems
OCTO    →  orchestrates     multiple arms, decision surface, human at the center
TRACE   →  records          append-only execution memory, every arm and chain
RECALL  →  publishes        structured documents, sovereign HTML
Mere    →  contains         the file is the app
```

TRACE is the memory layer for REACH + OCTO — every arm, every chain, every human decision, recorded immutably as it happens. See [semanticintent/trace](https://github.com/semanticintent/trace) (DOI: [10.5281/zenodo.20739404](https://doi.org/10.5281/zenodo.20739404)).

---

## Requirements

- .NET 10 SDK
- Windows (COM Interop, Task Scheduler, WinForms — deliberately local, no cloud)
- Claude Code CLI (or any Claude interface)
- Playwright browser binaries — `playwright install chromium` for browser reaches

---

## Citation

```bibtex
@misc{shatny2026reach,
  author    = {Shatny, Michael},
  title     = {REACH: Runtime Executable Adaptive C\# Handler — A Local Workflow Practice},
  year      = {2026},
  publisher = {Zenodo},
  doi       = {10.5281/zenodo.20680385},
  url       = {https://reach.semanticintent.dev}
}
```

---

## License

MIT — [Michael Shatny](https://semanticintent.dev)
