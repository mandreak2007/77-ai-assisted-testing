# Requirements Traceability Matrix (RTM) — Excel Guide

## Overview

The master-rtm.xlsx is a **single, persistent workbook** that accumulates
every requirement ever processed. It is never recreated — only appended to.
It has **5 sheets** in this fixed order.

---

## Workbook Sheet Order

1. `Version History`  — Audit log of every processing session
2. `RTM`              — Main requirements-to-scenarios matrix
3. `Scenarios`        — Full scenario inventory
4. `Duplicates`       — Log of all duplicate/partial-duplicate findings
5. `Coverage Summary` — Live statistics dashboard

---

## Sheet 1: Version History

This is the **first sheet** — it is the audit trail of the file.
Every time a BRD is processed, one new row is appended here.

### Column Definitions

| Col | Header              | Description                                               |
|-----|---------------------|-----------------------------------------------------------|
| A   | Version             | Version string: v1.0, v1.1, v2.0, ... (see rules below)  |
| B   | Date                | Session date (YYYY-MM-DD)                                 |
| C   | BRD / Module        | Name or source of the BRD processed this session         |
| D   | Reqs Added (Unique) | Count of new unique requirements added                    |
| E   | Reqs Duplicated     | Count of duplicate requirements detected                  |
| F   | Reqs Partial Dup    | Count of partial duplicate requirements detected          |
| G   | Scenarios Added     | Count of new scenarios added to Scenarios sheet           |
| H   | Total Reqs (All-time)| Running total of all rows in RTM sheet                   |
| I   | Total Scenarios (All-time) | Running total of all rows in Scenarios sheet       |
| J   | Changed By          | "QA Test Analyst Skill" or user-specified name            |
| K   | Summary of Changes  | Free-text summary of what was added/changed/flagged       |

### Version Numbering Rules

- **Major version** (v1.0 → v2.0): New feature module or BRD that adds 10+
  requirements, or a complete re-baseline of the project.
- **Minor version** (v1.0 → v1.1): Any session that adds new requirements
  or scenarios, or detects new duplicates.
- **Patch version** (v1.1 → v1.1.1): Corrections to existing rows
  (typo fixes, tag corrections, re-classifications).
- First run always creates **v1.0**.

### Formatting Rules

- Header row: Dark blue `1F3864`, white bold 11pt Arial, height 30, center
- Data rows: Alternating white `FFFFFF` / light gray `F2F2F2`, 10pt Arial
- Column A (Version): Bold, centered, light blue background `D9E1F2`
- Column B (Date): Centered, format as YYYY-MM-DD text
- Freeze pane at row 2
- Auto-filter on header row
- Column widths: A=10, B=14, C=25, D=18, E=18, F=18, G=18, H=20, I=22, J=20, K=50

---

## Sheet 2: RTM

### Column Definitions

| Col | Header                | Description                                              |
|-----|-----------------------|----------------------------------------------------------|
| A   | Req ID                | Requirement identifier (REQ-001, REQ-002, ...)           |
| B   | Requirement Name      | Short title of the requirement                           |
| C   | Requirement Description | Full description                                       |
| D   | Priority              | High / Medium / Low                                      |
| E   | Status                | UNIQUE / DUPLICATE / PARTIAL DUPLICATE                   |
| F   | Traced To REQ-ID      | For DUPLICATE/PARTIAL: the original REQ-ID it maps to   |
| G   | BRD Source            | Name of the BRD this requirement came from               |
| H   | Date Added            | YYYY-MM-DD when this requirement was first processed     |
| I   | Scenario ID           | Linked TC-ID (blank for full duplicates)                 |
| J   | Scenario Title        | Title of the linked BDD scenario                         |
| K   | Scenario Type         | Positive / Negative / Boundary / Business Rule           |
| L   | Tags                  | Gherkin tags                                             |
| M   | Test Status           | Not Executed (default)                                   |
| N   | Remarks               | Notes, assumptions, or gap descriptions                  |

### Formatting Rules

**Header row (row 1):**
- Background: Dark blue `1F3864`, font white bold 11pt Arial
- Alignment: Center, wrap text, row height 30

**UNIQUE requirement rows:**
- Req columns (A–H) grouped bg: alternating `D9E1F2` / `EBF0FA`
- Scenario columns (I–N): white `FFFFFF` / `F7F9FD`

**DUPLICATE requirement rows:**
- Entire row background: Light red `FFF0F0`
- Column E (Status) cell: Red fill `FFC7CE`, red bold font `9C0006`, text "DUPLICATE"
- Column F (Traced To): Orange fill `FFE699`, bold, must contain original REQ-ID
- Tooltip / remark: "Duplicate of [REQ-XXX] — no scenarios created"

**PARTIAL DUPLICATE requirement rows:**
- Entire row background: Light yellow `FFFDE7`
- Column E (Status) cell: Orange fill `FFE699`, dark font `7F6000`, text "PARTIAL DUPLICATE"
- Column F (Traced To): Yellow fill, must contain related REQ-ID
- Remark: "Partial duplicate of [REQ-XXX] — net-new scenarios only"

**Priority column (D) color coding:**
- "High": `FFC7CE`   "Medium": `FFEB9C`   "Low": `C6EFCE`

**Status column (M) color coding:**
- "Not Executed": `D9D9D9`   "Pass": `C6EFCE`   "Fail": `FFC7CE`
- "Blocked": `FFE699`        "In Progress": `FFEB9C`

**Borders:** Thin border around all cells. Medium border separating req groups.
**Freeze panes:** Row 1 + column A.
**Auto-filter:** On header row.

### Column widths:
A=12, B=25, C=45, D=10, E=18, F=18, G=22, H=14, I=12, J=45, K=18, L=32, M=15, N=40

---

## Sheet 3: Scenarios

### Column Definitions

| Col | Header           | Description                                           |
|-----|------------------|-------------------------------------------------------|
| A   | Scenario ID      | Unique TC-ID (TC-001, TC-002, ...) — never reused     |
| B   | Req ID           | Linked requirement ID                                 |
| C   | Feature File     | Name of the .feature file containing this scenario    |
| D   | Scenario Title   | Full scenario title                                   |
| E   | Scenario Type    | Positive / Negative / Boundary / Business Rule        |
| F   | Tags             | All Gherkin tags                                      |
| G   | Given            | Given step(s) summary                                |
| H   | When             | When step summary                                     |
| I   | Then             | Then step(s) summary                                 |
| J   | Test Data        | Key test data values used                             |
| K   | Priority         | High / Medium / Low                                   |
| L   | Execution Status | Not Executed (default)                                |
| M   | BRD Source       | Which BRD this scenario was created from              |
| N   | Date Added       | YYYY-MM-DD when this scenario was created             |
| O   | RTM Version      | Version string at time of creation (e.g., v1.2)       |

### Formatting Rules

- Header: Same dark blue as RTM sheet
- Data rows alternating: `FFFFFF` / `F2F2F2`, 10pt Arial, row height 18
- Wrap text all cells
- Freeze pane at row 2, auto-filter on header row
- Column widths: A=12, B=10, C=22, D=40, E=16, F=28, G=35, H=35, I=35, J=30, K=10, L=16, M=22, N=14, O=12

---

## Sheet 4: Duplicates

This sheet is the **full log of every duplicate finding** across all sessions.
One row per detected DUPLICATE or PARTIAL DUPLICATE requirement.

### Column Definitions

| Col | Header              | Description                                             |
|-----|---------------------|---------------------------------------------------------|
| A   | Detection Date      | YYYY-MM-DD when the duplicate was detected              |
| B   | RTM Version         | Version at time of detection                            |
| C   | Incoming REQ-ID     | The REQ-ID of the newly submitted (duplicate) req       |
| D   | Incoming Req Name   | Name of the incoming requirement                        |
| E   | Incoming Req Desc   | Full description of the incoming requirement            |
| F   | Duplicate Type      | DUPLICATE / PARTIAL DUPLICATE                           |
| G   | Matched REQ-ID      | Original REQ-ID that the incoming req duplicates        |
| H   | Matched Req Name    | Name of the original matched requirement                |
| I   | Match Signals       | Which signals triggered: name / description / rule / actor / outcome |
| J   | Match Signal Count  | Number of signals matched (3–5 = DUPLICATE, 1–2 = PARTIAL) |
| K   | Action Taken        | "No scenarios created" / "Net-new scenarios TC-XXX added" |
| L   | BRD Source          | BRD where the duplicate came from                       |
| M   | Remarks             | Any additional analyst notes                            |

### Formatting Rules

- Header: Dark blue `1F3864`, white bold, row height 30
- DUPLICATE rows: Light red background `FFF0F0`
- PARTIAL DUPLICATE rows: Light yellow background `FFFDE7`
- Column F (Duplicate Type):
  - "DUPLICATE": Red fill `FFC7CE`, red bold font `9C0006`
  - "PARTIAL DUPLICATE": Orange fill `FFE699`, dark font `7F6000`
- Column J (Match Signal Count): Centered, bold
  - 3–5: Red font `C00000`
  - 1–2: Orange font `E26B0A`
- Freeze pane at row 2, auto-filter on header
- Column widths: A=14, B=12, C=14, D=25, E=45, F=18, G=14, H=25, I=45, J=16, K=40, L=22, M=35

---

## Sheet 5: Coverage Summary

Live dashboard. All values use Excel formulas — never hardcode.

### Section 1: Overview (rows 2–10)
Title: "📊 Test Coverage Summary" — merged B2:F2, dark blue bg, white bold 14pt

| Metric                          | Formula                                         |
|---------------------------------|-------------------------------------------------|
| Total Requirements (All-time)   | =COUNTA(RTM!A:A)-1                              |
| Unique Requirements             | =COUNTIF(RTM!E:E,"UNIQUE")                      |
| Duplicate Requirements          | =COUNTIF(RTM!E:E,"DUPLICATE")                   |
| Partial Duplicate Requirements  | =COUNTIF(RTM!E:E,"PARTIAL DUPLICATE")           |
| Total Scenarios (All-time)      | =COUNTA(Scenarios!A:A)-1                        |
| High Priority Requirements      | =COUNTIF(RTM!D:D,"High")                        |
| Medium Priority Requirements    | =COUNTIF(RTM!D:D,"Medium")                      |
| Overall Coverage %              | =COUNTIF(RTM!E:E,"UNIQUE")/MAX(1,COUNTA(RTM!A:A)-1) |

### Section 2: Scenario Type Breakdown (rows 12–18)
Title: "📋 Scenario Type Breakdown"

| Type          | Formula                                       |
|---------------|-----------------------------------------------|
| Positive      | =COUNTIF(Scenarios!E:E,"Positive")            |
| Negative      | =COUNTIF(Scenarios!E:E,"Negative")            |
| Boundary      | =COUNTIF(Scenarios!E:E,"Boundary")            |
| Business Rule | =COUNTIF(Scenarios!E:E,"Business Rule")       |
| Total         | =SUM(C13:C16)                                 |

### Section 3: Duplicate Summary (rows 20–24)
Title: "⚠ Duplicate Detection Summary"

| Metric              | Formula                                       |
|---------------------|-----------------------------------------------|
| Full Duplicates     | =COUNTIF(Duplicates!F:F,"DUPLICATE")          |
| Partial Duplicates  | =COUNTIF(Duplicates!F:F,"PARTIAL DUPLICATE")  |
| Total Flagged       | =SUM(C21:C22)                                 |
| Scenarios Saved     | =COUNTIF(Duplicates!K:K,"No scenarios created") |

### Section 4: Execution Status (rows 26–32)
Title: "🚦 Execution Status"

| Status       | Formula                                         |
|--------------|-------------------------------------------------|
| Not Executed | =COUNTIF(Scenarios!L:L,"Not Executed")          |
| Pass         | =COUNTIF(Scenarios!L:L,"Pass")                  |
| Fail         | =COUNTIF(Scenarios!L:L,"Fail")                  |
| Blocked      | =COUNTIF(Scenarios!L:L,"Blocked")               |
| In Progress  | =COUNTIF(Scenarios!L:L,"In Progress")           |

### Section 5: Version History (rows 34–36)
Title: "📌 Latest Version Info"

| Metric           | Formula                                          |
|------------------|--------------------------------------------------|
| Current Version  | Pull last non-empty value from Version History!A |
| Last Updated     | Pull last non-empty value from Version History!B |
| Last BRD Added   | Pull last non-empty value from Version History!C |

### Formatting Rules

- Background: White
- Section titles: Dark blue `1F3864`, white bold 12pt, merged B:F
- Label cells: Bold, gray bg `F2F2F2`, left-aligned, width B=35
- Value cells: Right-aligned, white bg, thin border, width C=15
- Coverage % cell: Format as `0.0%`; if = 100% apply green fill `C6EFCE`

---

## Python openpyxl Key Patterns

```python
from openpyxl import Workbook, load_workbook
from openpyxl.styles import Font, PatternFill, Alignment, Border, Side
from openpyxl.utils import get_column_letter

# Colors
DARK_BLUE  = "1F3864";  LT_BLUE = "D9E1F2";  ALT_BLUE = "EBF0FA"
WHITE      = "FFFFFF";  GRAY    = "F2F2F2"
DUP_RED    = "FFF0F0";  DUP_CELL = "FFC7CE";  DUP_FONT = "9C0006"
PART_YELL  = "FFFDE7";  PART_CELL = "FFE699"; PART_FONT = "7F6000"

# Load existing or create new
import os
RTM_PATH = "reports/master-rtm.xlsx"  # Claude Code; Claude.ai: /mnt/user-data/outputs/master-rtm.xlsx
if os.path.exists(RTM_PATH):
    wb = load_workbook(RTM_PATH)
    # Read existing state from sheets
    ws_versions  = wb["Version History"]
    ws_rtm       = wb["RTM"]
    ws_scenarios = wb["Scenarios"]
    ws_dupes     = wb["Duplicates"]
    ws_summary   = wb["Coverage Summary"]
else:
    wb = Workbook()
    # Create all 5 sheets in order
    ws_versions  = wb.active;  ws_versions.title  = "Version History"
    ws_rtm       = wb.create_sheet("RTM")
    ws_scenarios = wb.create_sheet("Scenarios")
    ws_dupes     = wb.create_sheet("Duplicates")
    ws_summary   = wb.create_sheet("Coverage Summary")

# Read highest existing IDs
def get_max_id(ws, col=1, prefix="REQ-"):
    max_n = 0
    for row in ws.iter_rows(min_row=2, max_col=col, values_only=True):
        val = row[col-1]
        if val and str(val).startswith(prefix):
            try: max_n = max(max_n, int(str(val).split("-")[1]))
            except: pass
    return max_n

# Border helpers
thin = Side(style="thin"); med = Side(style="medium")
def tborder(): return Border(left=thin, right=thin, top=thin, bottom=thin)

# Style helpers
def hdr_style(cell):
    cell.font      = Font(name="Arial", bold=True, color="FFFFFF", size=11)
    cell.fill      = PatternFill("solid", start_color=DARK_BLUE)
    cell.alignment = Alignment(horizontal="center", vertical="center", wrap_text=True)
    cell.border    = tborder()
```

Always call `scripts/recalc.py` after saving to recalculate all formulas.
Verify zero formula errors before delivering to the user.
