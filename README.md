# TimePulse — Agentic Timesheet Compliance & Ops Reporting

MDT Group Project · Atharva Gupta (2501093)

## What this repository is

TimePulse is three autonomous Gen-AI agents that run a weekly timesheet-compliance
process end to end, with no human operator and no application server. There is
no traditional codebase to speak of — **the "source code" is the natural-language
agent instructions in `agents/*/SKILL.md`.** Each file is the complete, literal
prompt an autonomous Claude agent runs on a schedule. The agent reads it, then
carries it out by calling live tools — no separate backend, database, or business
logic implemented in a general-purpose programming language.

Concretely, each agent is a **Claude scheduled task**. On its cron schedule, Claude
loads the matching `SKILL.md` as its instructions and autonomously calls **Zapier's
MCP (Model Context Protocol) connector** to read and write a Google Sheet and send
Gmail — the same three actions a human coordinator would otherwise do by hand every
Monday morning.

This is a deliberate architectural choice for this project: it demonstrates an
**agentic** Gen-AI intervention (an LLM making autonomous decisions and taking
real actions across turns) rather than a single prompt-response Gen-AI feature.

## The three agents

| Schedule (Mon) | Agent | What it does | Touches the sheet |
|---|---|---|---|
| 06:00 | **Compliance Sentinel** | Computes each employee's cumulative missing-timesheet count since tracking began, and — only when a *new* gap has appeared since the last run — sends a tiered reminder (Tier 1 → Tier 2 → Tier 3 with manager cc'd) | Reads + writes (`Defaulter List`) |
| 06:15 | **Approval Steward** | Finds every timesheet stuck in "Pending Approval" and sends each approver one consolidated reminder listing all of their pending items | Read-only |
| 06:30 | **Insight Reporter** | Summarizes current compliance status (on-track / defaulter / escalated counts, repeat offenders) into one leadership email | Read-only |

The 15-minute stagger is intentional: Insight Reporter reads numbers that
Compliance Sentinel writes, so it must run after it.

## Repository contents

```
agents/
  compliance-sentinel/SKILL.md   — the exact, current prompt Claude runs weekly
  approval-steward/SKILL.md      — the exact, current prompt Claude runs weekly
  insight-reporter/SKILL.md      — the exact, current prompt Claude runs weekly
```

Each `SKILL.md` is copied verbatim from the live scheduled task — this is the
literal, currently-running version, not a simplified or illustrative excerpt.

## How it actually works (architecture)

1. **Trigger.** Claude's scheduler fires each agent on its cron schedule (or it can
   be run on demand — "run the Compliance Sentinel agent now").
2. **Reasoning.** Claude reads that agent's `SKILL.md` as its task and reasons
   over it autonomously — deciding who to email, what tier, what to write back —
   with no human approving individual steps.
3. **Action.** Claude calls **Zapier MCP** tool actions to actually do the work:

   | App | Action | Used by |
   |---|---|---|
   | Google Sheets | Get Many Spreadsheet Rows (Advanced) | Sentinel, Steward, Reporter |
   | Google Sheets | Lookup Spreadsheet Rows (Advanced) | Steward, Reporter |
   | Google Sheets | Update Spreadsheet Row(s) | Sentinel |
   | Gmail | Send Email | all three |

4. **Data.** All three agents read/write one Google Sheet with three tabs:

   **Timesheet Extraction** — `EntryID, EmployeeID, EmployeeName, WeekEndingDate,
   SubmittedDate, ApprovalStatus, ApproverID, ApproverName, ApprovalDate`

   **Employee Information** — `EmployeeID, FullName, Email, Department, JobTitle,
   HierarchyLevel, ManagerID, ManagerName, ManagerEmail, TimeTrackingRequired,
   IsLeadershipRecipient`

   **Defaulter List** — `EmployeeID, EmployeeName, Email, ManagerName, ManagerEmail,
   TotalMissingWeeks, Status, LastChecked, LastEmailSentDate, LastEmailType`
   (maintained by Compliance Sentinel; read by Insight Reporter)

No other infrastructure exists — no server to host, no database to provision, no
frontend required for the agents themselves to function.

## Current status: TEST PHASE

Every email any agent sends is redirected to one fixed test inbox
(`mdt.group8@gmail.com`) instead of real employees/managers/leadership. The first
line of every email body still states who it *would* really have gone to (e.g.
"Would be sent to: Priya Nair <priya.nair@skandemo.com>"), so the logic is fully
demonstrable without emailing anyone for real. Flipping to live sending is a small,
deliberate edit to each prompt (drop the TEST-phase clause, point `to` at the real
address field, drop the "Would be sent to" line) — intentionally not done yet for
this submission.

## Reproducing this

1. Connect a Zapier MCP server to Claude with the five actions listed above
   enabled, and connect the Google account/sheet and the Gmail account to send from.
2. Create a Google Sheet with the three tabs and columns above. `ApprovalStatus`
   must contain the literal string `Pending Approval` for pending rows, and at
   least one Employee Information row needs `Y` in `IsLeadershipRecipient` —
   both are matched exactly.
3. Create three scheduled tasks in Claude, one per file in `agents/`, pasting each
   `SKILL.md` body in as the task's prompt, on the schedule noted in the table
   above.

Three values inside the prompts are specific to this project's own Zapier/Google
accounts and would need to be swapped for a fork: the Zapier MCP server id
(`d748e143-65f5-4723-987e-bf18fbdbc560`), the spreadsheet ID
(`1jc_XCOlkBzs3KUAoCe2HlhxYiMxe5MSQ6H3X4QdYidU`), and the three worksheet GIDs.
Everything else in each prompt is portable as-is.

## Note on the dataset

The sheet these agents run against is seeded with a fictitious dataset (invented
names, `@skandemo.com` addresses, synthetic timesheet history) built for this
project — no real personal or organizational data is involved anywhere in this
system.
