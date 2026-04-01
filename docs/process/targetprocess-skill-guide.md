# Targetprocess Skill — Setup & Usage Guide

**For:** MinuteMenu Dev & QA Team
**Works with:** Claude Code, GitHub Copilot
**Updated:** April 2026

---

## Quick Summary

The Targetprocess (TP) skill lets your AI agent interact with TP directly — get tickets, create test cases, run tests, upload screenshots, post comments — all from your editor.

**Use this instead of the MCP server.** The skill calls the TP REST API via lightweight bash scripts. No server process, no token in every request, and it covers more operations (19 total vs 11 in MCP).

```
                MCP Server                          Skill (recommended)
  ┌──────────────────────────┐        ┌──────────────────────────────────┐
  │ Runs as separate process │        │ No process needed                │
  │ 11 tools                 │        │ 19 operations                    │
  │ Token passed per request │        │ Token auto-read from MCP config  │
  │ Limited to MCP protocol  │        │ Full REST API access             │
  │ No test case management  │        │ Full test lifecycle              │
  └──────────────────────────┘        └──────────────────────────────────┘
```

---

## 1. Setup

### Get the latest skill

The skill lives in the team repo: **https://github.com/MinuteMenu/mm-agent-skills**

Pull the latest and copy the `targetprocess` skill to your Claude Code skills folder:

```bash
git clone https://github.com/MinuteMenu/mm-agent-skills.git   # first time only
cd mm-agent-skills && git pull origin main                      # get latest
cp -r skills/targetprocess/* ~/.claude/skills/targetprocess/    # install for Claude Code
```

For GitHub Copilot, also copy to your project:

```bash
cp -r skills/targetprocess/* .github/copilot/skills/targetprocess/
```

### Token — no extra setup needed

The skill automatically reads your TP token from `~/.claude.json` where your existing TP MCP server is configured (`mcpServers.targetprocess.headers.X-TP-Token`). Since everyone already has the TP MCP set up, no additional token configuration is required.

### Install Office Viewer extension in VS Code

Install the **Office Viewer** extension (id: `cweijan.vscode-office`) from the VS Code marketplace.

```
Extensions (Ctrl+Shift+X) → Search "Office Viewer" → Install
```

**Why:** Test cases are pulled as markdown files with step tables. Office Viewer lets you edit these tables visually instead of wrestling with raw markdown pipe characters. This is essential for the test case review workflow.

---

## 2. Key Features

| Feature             | What it does                               | Example prompt                                                                                            |
| ------------------- | ------------------------------------------ | --------------------------------------------------------------------------------------------------------- |
| Get ticket          | Fetch full ticket details with comments    | "using targetprocess skills get ticket 318945"                                                            |
| Post comment        | Add comment to a ticket (supports images)  | "post this analysis comment to the ticket 318945"                                                         |
| Upload screenshot   | Attach a screenshot to a ticket            | "using targetprocess skills upload screenshot to ticket 318945"                                           |
| Create test cases   | Create test plan + test cases with steps   | "using targetprocess skills upload above test cases to ticket 318945"                                     |
| Pull test cases     | Download test cases as markdown for review | "using targetprocess skills download test plan of ticket 318945"                                          |
| Push test changes   | Update test cases from edited markdown     | "I have reviewed and updated some test cases in the test plan, push changes from ./test-cases/318945.md"  |
| Create test run     | Start a test run for a ticket              | "using targetprocess skills create test run for ticket 318945"                                            |
| Update test results | Mark tests passed/failed with step details | "using targetprocess skills create a test run and mark TC 1-9 passed, TC 10 failed at step 3"             |
| Attach to test case | Upload screenshot to test case evidence    | Handled automatically during result update                                                                |
| Tag test cases      | Organize tests into suites via tags        | "using targetprocess skills assign tag regression to all test cases in ticket 318945"                     |
| Search by tag       | Find tests across projects by tag          | "using targetprocess skills find all test cases with regression and eform tags"                           |
| Run tests by tag    | Create a run from tagged test cases        | "using targetprocess skills create a test run for all test cases tagged regression in kk support project" |

---

## 3. Test Case Review Workflow

This is the core workflow for QA reviewing dev-written test cases.

```
  Dev writes test cases          QA reviews in VS Code           QA pushes back to TP
  on the ticket in TP            with Office Viewer              with change summary
  ┌─────────────────┐           ┌─────────────────┐            ┌─────────────────┐
  │ Agent creates    │           │ Pull to local    │            │ Push changes     │
  │ test cases with  │──────────>│ Edit in markdown │──────────>│ Review diff      │
  │ steps on ticket  │           │ Add/remove/edit  │            │ Confirm update   │
  │                  │           │ steps            │            │ Comment posted   │
  └─────────────────┘           └─────────────────┘            └─────────────────┘
```

### Step 1 — Pull test cases to local

```
using targetprocess skills download test plan of ticket 318945 to ./test-cases/318945.md
```

This creates a markdown file with all test cases in a clean, editable format:

```markdown
# Test Plan: Fix Child Income Eligibility (ID: 319211)

**Ticket:** https://minutemenu.tpondemand.com/entity/318945
**Test Plan ID:** 319211
**Test Cases:** 15

---

## TC-319212: Verify checkbox is auto-checked for own child

**Tags:** (none) | **Priority:** Moderate | **Last Run:** Passed

| # | Step | Expected Result |
|---|------|-----------------|
| 1 | Log in KK by HX sponsor | |
| 2 | Go to Children > Enroll Child | |
| 3 | Select Tier 1 by income provider, own child | |
| 4 | Observe the Rule tab | Checkbox is checked, dates match |
```

### Step 2 — Review and edit in VS Code

Open the file in VS Code. Use **Office Viewer** to get a visual table editor:

- Right-click the `.md` file → **Open With...** → **Office Viewer**
- Or use the command palette: `Open Office Viewer`

**Common edits:**

- Fix unclear step descriptions
- Add missing expected results
- Add new test cases (use `## TC-NEW: Name` heading)
- Add tags (change `(none)` to `regression, smoke`)
- Change priority
- Remove irrelevant test cases

### Step 3 — Push changes back to TP

```
I have reviewed and updated some test cases in the test plan, push changes from ./test-cases/318945.md
```

The agent will:

1. **Show you a diff** of what changed (side-by-side comparison)
2. **Ask for confirmation** before applying anything
3. **Update TP** — only changed test cases are modified
4. **Post a comment** on the ticket summarizing all changes with your name

Example diff output:

```
## Changes Detected (2 of 15 test cases modified)

### TC-319212: Verify checkbox is auto-checked for own child
  **Tags:** (none) → regression
  **Step 4 updated:**
    Expected: "Checkbox is checked" → "Checkbox is checked, dates match provider's dates"

### TC-NEW: Verify edge case with NULL enrollment
  (new test case — will be created)
  3 steps defined
```

---

## 4. Running Tests and Updating Results

### Create a test run

```
using targetprocess skills create test run for ticket 318945
```

This creates a TestPlanRun with a TestCaseRun for each test case. The agent returns a list with Case Run IDs.

### Update results after testing

```
using targetprocess skills update test results for run 319417:
mark TC 26134 through 26142 as passed,
mark TC 26143 as failed at step 3 with comment "dropdown not populated on QA1 build sponsor_sec_20260401"
```

The agent updates:

- Each **TestCaseRun** overall status (Passed/Failed)
- Each **TestStepRun** within the case (step-level pass/fail)
- Attaches screenshots if provided

### Run tests by tag (regression suite)

```
using targetprocess skills create a test run for all test cases tagged regression in kk support project
```

This searches for all test cases with the "regression" tag, creates an ad-hoc test plan, and starts a run.
