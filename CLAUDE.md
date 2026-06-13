# REACH — Runtime Executable Adaptive C# Handler

REACH is a local workflow practice where Claude authors single-file C# scripts on demand
using `dotnet run task.cs`. No pre-built tools. No fixed implementations. Claude picks the
right library for the task, writes typed compiled code, runs it, reads the output, and iterates.

Each reach is ephemeral — written for the moment, executed, discarded. Nothing accumulates.
The capability surface is unlimited because Claude chooses the library, not you.

---

## How a Reach Works

```
dotnet run task.cs              # execute a reach
dotnet run task.cs --arg value  # with arguments
```

Each reach declares its own dependencies inline — no `.csproj`, no scaffolding:

```csharp
#:package Microsoft.Data.SqlClient@5.2.0
#:package DocumentFormat.OpenXml@3.0.0
#:sdk Microsoft.NET.Sdk
```

Claude writes the file, runs it, reads stdout/stderr, fixes compile errors, reruns.
Compile errors are a feature — they surface immediately and are typed. Claude fixes them
faster than debugging silent PowerShell failures.

---

## Reach Index

| Domain | Task | Key Library |
|---|---|---|
| Screen | Capture window or region | `System.Drawing` (built-in) |
| Screen | Capture specific process window | `System.Drawing` + P/Invoke |
| Email | Read Outlook, save body + attachments | `Microsoft.Office.Interop.Outlook` |
| Documents | Extract text from Word | `DocumentFormat.OpenXml` |
| Documents | Extract text from PDF | `PdfPig` |
| Database | Query SQL Server environments | `Microsoft.Data.SqlClient` |
| Database | Query and output as JSON/CSV | `Microsoft.Data.SqlClient` + `System.Text.Json` |
| Files | Watch folder, react to changes | `System.IO.FileSystemWatcher` (built-in) |
| Files | Merge multiple text files into one | `System.IO` (built-in) |
| Browser | Scrape static HTML page content | `HtmlAgilityPack` |
| Browser | Headless browser — JS pages, SPAs, auth flows | `Microsoft.Playwright` |
| Browser | Screenshot specific page or element | `Microsoft.Playwright` (built-in) |
| Browser | Interact — click, fill forms, navigate | `Microsoft.Playwright` |
| OS | Launch applications, send keystrokes | `System.Diagnostics` + P/Invoke |
| OS | UI automation (click, read controls) | `FlaUI` |
| HTTP | Call REST APIs | `System.Net.Http` (built-in) |
| Output | Format and write structured reports | `System.Text.Json`, `CsvHelper` |

---

## Reaches — Detail

---

### Screenshots

**When:** Visual context needed — UI review, deployment state, error on screen, anything
that words don't capture efficiently. Claude reaches for this itself when it determines
it needs to see something, without you having to take the screenshot manually.

**Claude decides:** full screen, named window, or defined region — based on what the task needs.

```csharp
#:sdk Microsoft.NET.Sdk

using System.Drawing;
using System.Drawing.Imaging;

var path = Path.Combine(Path.GetTempPath(), $"reach_snap_{DateTime.Now:yyyyMMdd_HHmmss}.png");
using var bmp = new Bitmap(Screen.PrimaryScreen.Bounds.Width, Screen.PrimaryScreen.Bounds.Height);
using var g = Graphics.FromImage(bmp);
g.CopyFromScreen(Point.Empty, Point.Empty, bmp.Size);
bmp.Save(path, ImageFormat.Png);
Console.WriteLine(path);  // Claude reads this path back and loads the image inline
```

**Performance:** 3-5s cold, under 1s warm. Fastest reach available.

**Trigger phrases:**
- "review the staging UI in Chrome"
- "snap the current browser window"
- "screenshot the error on screen"

Claude captures, reads the image inline, and reasons about what it sees — you never touch it.

---

### Email — Save Body and Extract Attachments

**When:** Email arrives with embedded content or attachments. Need to save body as text,
pull attachments to disk for further processing — feeding into context, extraction, analysis.

**Library:** `Microsoft.Office.Interop.Outlook` — talks directly to your running Outlook
instance via COM. No API keys, no Graph auth, no tokens. Works because Outlook is already open.

```csharp
#:package Microsoft.Office.Interop.Outlook@15.0.4797.1004

using Outlook = Microsoft.Office.Interop.Outlook;

var app = new Outlook.Application();
var explorer = app.ActiveExplorer();
var mail = (Outlook.MailItem)explorer.Selection[1]; // currently selected email

// Save body
File.WriteAllText("email_body.txt", mail.Body);

// Save attachments
Directory.CreateDirectory("attachments");
foreach (Outlook.Attachment att in mail.Attachments)
{
    att.SaveAsFile(Path.Combine("attachments", att.FileName));
    Console.WriteLine($"Saved: {att.FileName}");
}
```

**Trigger phrases:**
- "save this email and its attachments"
- "extract everything from the selected email"
- "pull the attachments out and list them"

---

### Word Documents — Extract Text

**When:** Attachment is a `.docx` and you need its content as plain text — to feed into
context, summarize, compare against something else, or save alongside other materials.

**Library:** `DocumentFormat.OpenXml` — official Microsoft library, handles all Word internals.

```csharp
#:package DocumentFormat.OpenXml@3.0.0

using DocumentFormat.OpenXml.Packaging;
using DocumentFormat.OpenXml.Wordprocessing;

using var doc = WordprocessingDocument.Open("document.docx", false);
var body = doc.MainDocumentPart.Document.Body;
var text = string.Join("\n", body.Descendants<Paragraph>().Select(p => p.InnerText));
File.WriteAllText("document.txt", text);
Console.WriteLine($"Extracted {text.Length} characters");
```

**Trigger phrases:**
- "extract text from this Word doc"
- "save the content of document.docx as text"
- "pull the readable content out of the attachment"

---

### PDF Documents — Extract Text

**When:** Attachment or reference is a PDF. Need readable content without opening Acrobat.

**Library:** `PdfPig` — clean, modern, no native dependencies. Claude reaches for
`iTextSharp` instead if layout or structure matters more than raw text.

```csharp
#:package PdfPig@0.1.8

using UglyToad.PdfPig;
using UglyToad.PdfPig.Content;

using var pdf = PdfDocument.Open("document.pdf");
var sb = new System.Text.StringBuilder();
foreach (var page in pdf.GetPages())
    sb.AppendLine(string.Join(" ", page.GetWords().Select(w => w.Text)));

File.WriteAllText("document.txt", sb.ToString());
Console.WriteLine($"Extracted {pdf.NumberOfPages} pages");
```

**Trigger phrases:**
- "extract the text from this PDF"
- "save the PDF content as readable text"

---

### Database Queries — SQL Server

**When:** Need to query data across environments (dev, staging, prod). Output shaped for
whatever comes next — inspection, comparison, feeding back into conversation.

**Library:** `Microsoft.Data.SqlClient` — modern, actively maintained, works with all SQL
Server versions and Azure SQL.

**Claude decides:** output format based on context — JSON for structured review, CSV for
Excel handoff, plain text for quick inspection.

```csharp
#:package Microsoft.Data.SqlClient@5.2.0
#:package System.Text.Json@8.0.0

using Microsoft.Data.SqlClient;
using System.Text.Json;

var environments = new Dictionary<string, string>
{
    ["dev"]     = "Server=your-dev-server;Database=YourDatabase;Integrated Security=true;",
    ["staging"] = "Server=your-staging-server;Database=YourDatabase;Integrated Security=true;",
    ["prod"]    = "Server=your-prod-server;Database=YourDatabase;Integrated Security=true;"
};

var env = args.FirstOrDefault() ?? "staging";
var query = "SELECT TOP 100 * FROM Equipment WHERE Status = 'Active'";

using var conn = new SqlConnection(environments[env]);
await conn.OpenAsync();
using var cmd = new SqlCommand(query, conn);
using var reader = await cmd.ExecuteReaderAsync();

var results = new List<Dictionary<string, object>>();
while (await reader.ReadAsync())
{
    var row = new Dictionary<string, object>();
    for (int i = 0; i < reader.FieldCount; i++)
        row[reader.GetName(i)] = reader.IsDBNull(i) ? null : reader.GetValue(i);
    results.Add(row);
}

Console.WriteLine(JsonSerializer.Serialize(results, new JsonSerializerOptions { WriteIndented = true }));
```

**Trigger phrases:**
- "query staging for the projection issue in Equipment"
- "pull active records from prod and compare with staging"
- "run this against dev and show me the results"

---

### UI Automation — Click, Read, Navigate

**When:** Need to interact with a running application's UI — Outlook folders, a desktop app,
a form — without using its API. Reads the actual Windows UI control tree.

**Library:** `FlaUI` — wraps Windows UIAutomation cleanly. Finds controls by name, type,
or automation ID. Clicks buttons, reads list items, types into fields.

```csharp
#:package FlaUI.Core@4.0.0
#:package FlaUI.UIA3@4.0.0

using FlaUI.Core;
using FlaUI.Core.AutomationElements;
using FlaUI.UIA3;

using var automation = new UIA3Automation();
var app = Application.Attach("OUTLOOK");
using var window = app.GetMainWindow(automation);

// Claude writes specific interaction logic per task
var inbox = window.FindFirstDescendant(cf => cf.ByName("Inbox"));
Console.WriteLine(inbox?.Name ?? "not found");
```

**Trigger phrases:**
- "navigate to the Inbox in Outlook and list unread items"
- "click the filter button in the open application"
- "read the items in this list"

---

### File Operations — Merge, Watch, Organize

**When:** Multiple text outputs need combining into one file, or a folder needs watching
for new files to process automatically.

**Library:** Built-in `System.IO` — no packages needed. Claude writes exact logic for
the shape of the task.

```csharp
// Merge multiple extracted text files into one context file
var files = Directory.GetFiles("attachments", "*.txt");
var merged = string.Join("\n\n---\n\n", files.Select(File.ReadAllText));
File.WriteAllText("merged_context.txt", merged);
Console.WriteLine($"Merged {files.Length} files → merged_context.txt");
```

**Trigger phrases:**
- "merge all the extracted text files into one"
- "combine these documents into a single context file"
- "watch the attachments folder and tell me when something arrives"

---

### Browser — Headless (Playwright)

**When:** Page requires JavaScript to render, interaction with a SPA, navigating through
auth, waiting for dynamic content, or screenshotting a specific page state. Playwright
spins up a real headless Chromium — sees exactly what a browser sees.

**Library:** `Microsoft.Playwright` — pre-installed on this machine (`playwright install chromium`
already run). Claude picks Chromium by default; Firefox or WebKit if the task calls for it.

**Performance:** 10-20s cold, 6-10s warm. Heavier than screen capture but sees real pages.

```csharp
#:package Microsoft.Playwright@1.44.0

using Microsoft.Playwright;

using var playwright = await Playwright.CreateAsync();
await using var browser = await playwright.Chromium.LaunchAsync(new() { Headless = true });
var page = await browser.NewPageAsync();

await page.GotoAsync("https://internal-dashboard/deployments");
await page.WaitForSelectorAsync(".deployment-status");

// Extract text
var status = await page.InnerTextAsync(".deployment-status");
Console.WriteLine(status);

// Screenshot whole page or specific element
await page.ScreenshotAsync(new() { Path = "page.png", FullPage = true });
await page.Locator(".data-table").ScreenshotAsync(new() { Path = "table.png" });
```

**Auth flows** — Playwright fills login forms, handles redirects, persists session cookies:

```csharp
await page.GotoAsync("https://internal-tool/login");
await page.FillAsync("#username", "myuser");
await page.FillAsync("#password", "mypass");
await page.ClickAsync("button[type=submit]");
await page.WaitForURLAsync("**/dashboard");
// authenticated — continue navigating
```

**Trigger phrases:**
- "open the staging dashboard and screenshot the deployment table"
- "check the internal portal and extract the current queue status"
- "navigate to the report page and pull the numbers from the table"
- "log into the tool and grab the latest entries"

---

### Browser — Static HTML (HtmlAgilityPack)

**When:** Page is plain HTML — no JavaScript rendering needed. Faster and lighter than
launching a full browser process. Good for internal pages, documentation sites, or any
URL where `curl` would return the content you need.

**Library:** `HtmlAgilityPack` — HTTP fetch + full DOM parsing with XPath or CSS selectors.
No browser process, near-instant startup.

```csharp
#:package HtmlAgilityPack@1.11.61

using HtmlAgilityPack;

var web = new HtmlWeb();
var doc = web.Load("https://internal-wiki/release-notes");

// XPath selection
var rows = doc.DocumentNode.SelectNodes("//table[@class='releases']//tr");
foreach (var row in rows ?? Enumerable.Empty<HtmlNode>())
    Console.WriteLine(row.InnerText.Trim());
```

**Trigger phrases:**
- "pull the content from this internal wiki page"
- "extract the table from the release notes page"
- "scrape the text from this URL"

---

### HTTP / REST Calls

**When:** Need to hit an internal API, webhook, or external service — check a deployment
status endpoint, post a notification, pull JSON from a service.

**Library:** `System.Net.Http.HttpClient` — built-in, no package needed.

```csharp
using var http = new HttpClient();
var response = await http.GetStringAsync("https://internal-api/deployment/status");
Console.WriteLine(response);
```

**Trigger phrases:**
- "check the deployment status endpoint"
- "hit the staging health check and tell me what it returns"

---

## How Claude Chooses Libraries

Claude selects the least-friction library for the specific task. Built-ins first.

| Situation | Claude reaches for |
|---|---|
| File is `.docx` | `DocumentFormat.OpenXml` — official, handles all Word internals |
| File is `.pdf` | `PdfPig` for text, `iTextSharp` if layout matters |
| Outlook is open | COM Interop — no auth, talks to live instance directly |
| SQL Server query | `Microsoft.Data.SqlClient` — modern, works across all envs |
| UI interaction needed | `FlaUI` — wraps Windows UIAutomation cleanly |
| Screen capture | `System.Drawing` built-in — no package, fastest reach |
| Page needs JavaScript / SPA | `Microsoft.Playwright` — real headless Chromium |
| Page is static HTML | `HtmlAgilityPack` — fast, no browser process |
| Need to interact with browser | `Microsoft.Playwright` — click, fill, navigate |
| Screenshot a page or element | `Microsoft.Playwright` built-in screenshot |
| Simple HTTP / REST | `HttpClient` built-in — no package needed |
| Structured output | `System.Text.Json` built-in or `CsvHelper` for CSV |

**Browser decision shortcut:** if `curl` would return what you need → `HtmlAgilityPack`.
If not → `Playwright`.

---

## REACH Iteration Loop

Claude follows this loop for every reach:

```
1. Understand intent from conversation context
2. Write task.cs with appropriate #:package directives
3. dotnet run task.cs
4. Read stdout / stderr
5. Compile error  → fix types, rerun
6. Runtime error  → adjust logic, rerun
7. Wrong output   → reason about it, adjust query/logic, rerun
8. Return result or continue conversation with findings
```

---

## REACH Performance

| Reach type | Cold (first ever) | Warm (cached) |
|---|---|---|
| Screen capture (`System.Drawing`) | 3-5s | <1s |
| SQL query | 4-8s | 1-2s |
| Word / PDF extraction | 5-10s | 1-3s |
| Playwright page screenshot | 10-20s | 6-10s |

NuGet packages cache locally after first download. Scripts cache after first compile.
The iteration loop feels fast inside a conversation because Claude's reasoning between
steps is the dominant variable, not script execution time.

---

## Session Modes

**Structured prompt (readme.txt / semantic intent)** — use when the task has multiple
connected layers described upfront. Claude gets the full picture before writing any reach.
Good for: email + attachments + DB query + output format all wired together.

**Conversational kickoff** — use when the task shape is still emerging. Claude writes
small exploratory reaches, output informs the next step, context builds through dialogue.
Good for: "we have a projection issue, query Equipment" — Claude starts narrow and expands.

Both modes work. REACH suits conversational mode especially well because each reach can
use completely different libraries — the tool is never fixed, it adapts to where the
conversation goes.

---

## Notes

- Requires .NET 10 SDK installed locally
- Playwright browser binaries pre-installed — `playwright install chromium` already run.
  Claude can use `Microsoft.Playwright` in any reach without additional setup.
- Claude Code CLI must run locally (not sandboxed) to reach your desktop, COM objects,
  and local database connections
- Reaches are ephemeral — written for the task, discarded after. Nothing accumulates.
- `dotnet project convert task.cs` promotes any reach to a full project if it outgrows
  the single-file model

---

## REACH Patterns — Real Sessions

### Sprint Cadence Review

**What it produces:** A synthesized picture of work rhythm across a sprint — email cadence,
deployment activity, meeting density, commit history — combined into a single narrative that
reveals pace, pressure points, and whether a cool-down week is needed.

**Session mode:** Conversational kickoff. Each reach builds on the previous one. Claude holds
context across all sources and synthesizes at the end — you don't need to describe the full
picture upfront, it emerges through the conversation.

**How this session unfolded:**

**Step 1 — Prove the visual reach works**
Start by confirming Playwright headless screenshot is operational. Claude captures the screen,
verifies the output. Establishes confidence before touching real data.

```
"let's see if headless browser screenshot works"
→ Claude writes Playwright reach, captures screen, returns image path
→ Claude reads image inline, confirms it works
```

**Step 2 — Screenshot the live project environment**
Claude, already initialized against the project, suggests capturing the actual staging
page rather than a generic screen — it already has project context and knows what's relevant.

```
"can we capture the staging page"
→ Claude navigates Playwright to staging URL
→ Screenshots the live UI, reads it, connects it to existing project knowledge
```

**Step 3 — Connect to Outlook, establish email baseline**
First Outlook reach: simple count of a folder to confirm COM connection is live.
Deleted folder used as a sanity check — 600+ unread, 605 total (5 read). Connection confirmed.

```
"let's connect to Outlook and check the deleted folder count"
→ Claude writes Outlook COM reach
→ Returns: 600+ unread, 605 total
→ COM connection confirmed, Outlook is reachable
```

**Step 4 — Read project email cadence from start of year**
The core reach. Claude reads project-related emails from the start of the period and groups them by month,
painting a picture of communication cadence across the sprint without being asked to structure it
that way — Claude inferred the right framing from the intent.

```
"read the cadence of project emails from the beginning of this year"
→ Claude reaches into Outlook, filters by sender/subject pattern, groups by month
→ Returns month-by-month cadence: volume, timing, gaps, spikes
→ Claude narrates what the pattern means — not just counts
```

**Step 5 — Check meeting calendar**
Adds the meeting dimension. Claude reaches into calendar data to layer meeting density
alongside email cadence — same period, same framing.

```
"check the meetings calendar for the same period"
→ Claude reaches into Outlook calendar
→ Returns meeting frequency and clustering by week
→ Claude connects meeting load to email cadence already in context
```

**Step 6 — Pull git commit history**
Final data source. Commit history adds the actual delivery rhythm — when work shipped,
how consistently, where the gaps were. Claude also analyzes **time-of-day distribution**
across commits — separating core hours from after-hours to surface invisible overload.

```
"pull the git commit history"
→ Claude runs git log with date range, parses output
→ Returns commit cadence by week
→ Breaks down commits by time: core hours vs after-hours vs weekend
→ Calculates after-hours percentage across the sprint
→ Claude synthesizes across all three sources: email + meetings + commits
```

**After-Hours Benchmark:**

| After-hours commit % | Reading | What it signals |
|---|---|---|
| 0-15% | Healthy | Occasional late push, not a pattern |
| 15-25% | Watch | Creeping into personal time regularly |
| 25%+ | Redline | Work has structurally overflowed into after-hours |
| 30%+ | All red | Sustained boundary erosion — unsustainable |

**This sprint:** 30% of commits were after-hours. Structural overflow, not occasional.
Target range: **10-15%** — allows for the rare late push without it becoming the norm.

The after-hours number is often the most honest signal in the whole review. Email and
meetings can be managed or deferred. A commit at 10pm means the work itself demanded it.

**Step 7 — Claude synthesizes unprompted**
With all sources in context Claude painted the full picture — 10 week sprint, cadence across
every month, pressure points visible, and a clear conclusion: one week cool-down needed.
You didn't ask for a conclusion. Claude drew it from the pattern.

**Step 8 — RPM interpretation and deload recommendation**
When asked "how does that look?" Claude read the combined signal — email volume, meeting
density, commit frequency — and called it: **all red. High RPM sustained across the full
10 weeks.** No week dropped below redline. Engine didn't blow but it never wasn't at max.

The recommendation that followed: one **open valley week** — intentionally low-commitment,
no deliverables, open capacity — inserted after the sprint. Not a full stop. Still moving,
but well below redline. Effect: the tachometer drops back into green, which reframes the
entire 10 weeks retroactively. The story shifts from *sustained overload* to *strong run,
well-managed cool-down*. Same data. Different narrative. Different reality going forward.

```
"how does that look?"
→ Claude reads combined signal across all sources
→ Calls the RPM state: all red / high RPM / normal RPM / green
→ If red: recommends open valley week and explains what it normalizes
→ Reframes the sprint arc: not burned out, intentionally recovered
```

---

**RPM Scale — how Claude reads sprint intensity:**

| RPM State | Email + Meetings | Commits | After-hours % | What it means |
|---|---|---|---|---|
| 🟢 Green | Low-moderate, spaced | Steady cadence | 0-15% | Sustainable, healthy rhythm |
| 🟡 Yellow | Elevated, frequent | Spikes visible | 15-25% | Manageable — watch the trend |
| 🔴 Red | High volume, dense | Compressed | 25-30% | Running hot — recovery needed |
| 🔴🔴 All red | Max across all weeks | Overloaded | 30%+ | Redline sustained — open valley essential |

**After-hours target: 10-15%.** Allows for the occasional late push without it becoming
structural. At 30% the work has overflowed its container — the sprint owns your evenings,
not just your days. That's the number that doesn't show up in standup but shows up in the data.

**Deload / Open Valley Week — what it is:**
A deliberate low-RPM week after a sustained high-intensity sprint. Not vacation, not absence —
still present, still moving, but nothing critical scheduled. No hard deliverables. Open calendar.
Lower email expectation. Effect: the system recovers, the sprint arc reads as intentional rather
than exhausted, and the next sprint starts from green not from fumes.

---

**Sources tapped in this session:**

| Source | Library | What it contributed |
|---|---|---|
| Staging UI | `Microsoft.Playwright` | Visual state of the live environment |
| Outlook email | `Microsoft.Office.Interop.Outlook` | Communication cadence, volume, gaps |
| Outlook calendar | `Microsoft.Office.Interop.Outlook` | Meeting density and clustering |
| Git history | `System.Diagnostics` (git log) | Delivery rhythm, commit cadence |

---

**To rerun this pattern next sprint:**

```
"run a sprint cadence review — email, calendar, commits, same pattern as last time"
→ Claude knows the pattern from CLAUDE.md
→ Executes each reach in sequence
→ Reads RPM state across all sources
→ Calls green / yellow / red / all red
→ If red: recommends open valley week and reframes the arc
→ Compares to prior sprint if notes are available
→ Delivers the picture
```

The second run is faster than the first. The third faster still. The pattern is now documented
and repeatable — what emerged conversationally becomes a named reach.
