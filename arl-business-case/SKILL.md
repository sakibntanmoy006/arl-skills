---
name: arl-business-case
description: Generate a board-ready business case for an AKIJ Resource SBU (e.g. AEL, ACCL, AAFL, AEFL) for a given fiscal year, in HTML, by pulling actuals + budget from the DataMart MCP and strategy/budget documents from the Strategy MCP Drive library. Use when the user asks to "make a business case", "prepare a business case", or "build a case" for a named SBU and FY. Inputs: SBU name/code and FY (e.g. "AEL FY 25-26").
license: MIT
metadata:
  author: humaninside
  tags: business-case, sbu, html, datamart, strategy, arl, akij, board, financial-analysis
---

# arl-business-case — SBU Business Case from DataMart + Strategy MCP

Produce a **board-ready business case in HTML** for one AKIJ Resource SBU and one fiscal year,
by fusing two read-only data sources:

| Source | MCP server | What it gives |
|---|---|---|
| Financial actuals & budget | `ARL-DataMart_*` (BI DataMart, BU-scoped) | Actual P&L by FS component, monthly trend, profit-center (category) P&L, approved budget |
| Strategy / budget docs | `ARL-Strategy_drive_*` (Google Drive "AR- 05 Years Strategy 27-31") | 5-year strategy, ABP, Income Statement PDF — the plan the business case argues for/against |

## Invocation

The user names an **SBU** and a **FY year**, e.g.:

- `make a business case for AEL FY 25-26`
- `business case for ACCL, FY 2026-27`
- `make a business case of AAFL 25-26`

Accept SBU by name or code (Akij Essentials Ltd / AEL), and FY in any common form
(`25-26`, `2025-26`, `FY25-26`, `2026-27`). If only the SBU is given without a FY, ask which FY.
If the SBU code is ambiguous, resolve it against `tblBusinessUnit` first.

## Output

- One self-contained HTML file named `<SBUCODE>_Business_Case_FY<YY>-<YY>.html` (e.g. `AEL_Business_Case_FY25-26.html`)
- Written next to the session working directory (usually the user's home/current dir)
- Print-ready: inline CSS, tables, color-coded negatives/positives, cover, sections 1–10
- All figures in **BDT Crore** unless a source states otherwise

## Workflow

Run the steps in order. Every step ends when its numbers reconcile (see **Reconciliation**).

### STEP 1 — Resolve the SBU (DataMart)

Query `tblBusinessUnit` to pin the SBU:

```sql
SELECT intBusinessUnitId, strBusinessUnitCode, strBusinessUnitName,
       strBusinessUnitGroupName, strBusinessUnitSubGroupname
FROM dbo.tblBusinessUnit
WHERE strBusinessUnitName LIKE '%<name>%' OR strBusinessUnitCode = '<CODE>'
```

Record `intBusinessUnitId` and `strBusinessUnitCode` — every later query uses `intBusinessUnitId`.
Some SBUs share a name with their group (e.g. AEL = Akij Essentials Ltd). If the SBU isn't in
`tblBusinessUnit`, note it and continue with the code the user gave; flag in the doc that the
BU id could not be verified.

### STEP 2 — Derive the FY date window

Bangladesh FY = Jul 1 → Jun 30. Given `<YY1>-<YY2>`:

| FY | Start | End |
|---|---|---|
| 24-25 | 2024-07-01 | 2025-06-30 |
| 25-26 | 2025-07-01 | 2026-06-30 |
| 26-27 | 2026-07-01 | 2027-06-30 |

Compute both the **target FY** window and the **prior FY** window (for YoY trend).

### STEP 3 — Pull actuals (DataMart `tblISTransaction`)

**3a. Annual P&L by FS component** (target FY and prior FY):

```sql
SELECT intFSComponentId, strFSComponentName, SUM(numAmount) AS TotalAmount
FROM dbo.tblISTransaction
WHERE intBusinessUnitId = <BU> AND dteTransactionDate >= '<start>' AND dteTransactionDate <= '<end>'
GROUP BY intFSComponentId, strFSComponentName
ORDER BY intFSComponentId
```

**3b. Monthly revenue/COGS/finance-cost trend** (target FY):

```sql
SELECT YEAR(dteTransactionDate) Yr, MONTH(dteTransactionDate) Mn,
       SUM(CASE WHEN intFSComponentId=1  THEN -numAmount ELSE 0 END) Revenue,
       SUM(CASE WHEN intFSComponentId=20 THEN  numAmount ELSE 0 END) COGS,
       SUM(CASE WHEN intFSComponentId=6  THEN  numAmount ELSE 0 END) FinCost
FROM dbo.tblISTransaction
WHERE intBusinessUnitId=<BU> AND dteTransactionDate>='<start>' AND dteTransactionDate<='<end>'
GROUP BY YEAR(dteTransactionDate), MONTH(dteTransactionDate)
ORDER BY Yr, Mn
```

**3c. Profit-center (category) P&L** (target FY) — for the category table:

```sql
SELECT intProfitCenterId, strProfitCenterName, intFSComponentId, strFSComponentName, SUM(numAmount) Amt
FROM dbo.tblISTransaction
WHERE intBusinessUnitId=<BU> AND dteTransactionDate>='<start>' AND dteTransactionDate<='<end>'
  AND intFSComponentId IN (1,20,21)
GROUP BY intProfitCenterId, strProfitCenterName, intFSComponentId, strFSComponentName
ORDER BY intProfitCenterId, intFSComponentId
```

> **Sign convention:** DataMart records revenue and income as **negative**, costs as **positive**.
> When presenting, convert to absolute values and apply the sign yourself: revenue −, costs −, income +.
> Cross-check: for AEL FY25-26, `SUM` of FS1 was −32,347,460,872 → Revenue 3,234.75 Cr.

### STEP 4 — Pull budget (DataMart `fin.tblISbudget`)

```sql
SELECT strFiscalYear, intFSComponentId, strFSComponentName, SUM(numAmount) BudgetAmount
FROM fin.tblISbudget
WHERE intBusinessUnitId=<BU>
GROUP BY strFiscalYear, intFSComponentId, strFSComponentName
ORDER BY strFiscalYear, intFSComponentId
```

Filter to the target FY (`strFiscalYear` = `<YY1>-<YY2>`). Budget may lag actuals on some lines
(missing FS components is normal — e.g. AEL FY25-26 budget had no Manufacturing Overhead). Map
budget to the same FS component IDs as actuals for the variance table.

> **Budget sanity check:** if the budget revenue is wildly different from the strategy target
> (e.g. AEL FY25-26 budget 6,217 Cr vs strategy target ~2,400 Cr), **do not hide it** — report
> both and note the discrepancy in the doc. Flag discrepancies, don't smooth them.

### STEP 5 — Pull strategy & budget docs (Strategy MCP Drive)

Drive root: **"AR- 05 Years Strategy 27-31"** (`1bno3WJa452ehSL1Hu1764AjmX7vBdkGe`). Children:
- `AR Strategy 05 Years (FY 26-27 to FY 30-31)` (`178VoXrIeLJbiSgJGxcbsYxoY4YJNIuwV`) — per-SBU 5-year strategy folders
- `Business Plan & Budget` (`1UeX3KNjHWfZwD7mWn7bwLiZYJnusNlPc`) → `Strategy & Budget 2026-27` → `All SBU` (`1B1ISOupgAP3CL1QBC8XSTqy11-ez8hlc`) — per-SBU ABP/budget files
- `Coporate Strategy` (`1KkmgaWtbolTcYVGB9c3aDRbcxcmeaId9`) — corporate-level plan

Locate the SBU's folder by code/name:
1. `ARL-Strategy_drive_folder_tree` on `178VoXrIeLJbiSgJGxcbsYxoY4YJNIuwV` and on `1B1ISOupgAP3CL1QBC8XSTqy11-ez8hlc` to find the SBU folder.
2. Inside it, prefer in order: `*_5Year_Strategy*.xlsx`/`*.html`, then `*ABP*.xlsx`, then `*Income statement*.pdf`, then the master `*Business Plan*.pdf`.
3. `ARL-Strategy_drive_read_file` each file. **If the file is large** (HTML >~40KB, xlsx with many sheets), do not read it fully into your own context — delegate extraction to an **explore** subagent (Task tool) with instructions to return exact figures. Large PDFs often can't be parsed inline; use a subagent to download/parse and return numbers.

Extract from the strategy/ABP docs:
- The **target-vs-actual gap table** for the FY in question (KPIs, targets, actuals, root causes)
- **Strategic pillars / initiatives** and the **5-year financial plan** (revenue, GM%, EBITDA, finance cost, PAT by FY)
- **Category budget** for the next FY, **capex**, **working-capital targets**, **key assumptions**, **risk register**
- Any **market/competitive** context (CPM scores, market CAGR, competitors)

If the Drive has no folder for the SBU, fall back to: corporate strategy doc + DataMart only, and
say so in the doc (section 2).

### STEP 6 — Reconcile before writing

Every figure in the doc must trace to a source and the numbers must add up:

- Revenue = Σ(monthly revenue) = the FS1 actual (within rounding)
- GP = Revenue − COGS − MOH (FS1, 20, 21)
- EBITDA = GP − (Admin + Marketing + Selling + Logistics + Other Op) (FS3,4,22,25,5)
- PAT = EBITDA − Dep(7) − Fin(6) + NonOp(8) − Tax(9)
- Strategy doc's "ERP Verified actual" should equal the DataMart sum for the same FY/BU

If a line doesn't reconcile, re-run the query before proceeding. Do not write numbers you can't
reconcile; mark any unverifiable figure with a note.

### STEP 7 — Write the HTML

Use `templates/business-case.html` as the shell (it carries all CSS + the 10-section skeleton).
Fill every `<-- ... -->` placeholder with the reconciled data. Keep the section structure:

1. Executive Summary (lead paragraph + KPI cards + decision sought)
2. Scope, Sources & Method (table; note BU scope exclusions)
3. Strategic Context (corporate linkage, market, core tension callout)
4. FY actuals: 4.1 P&L vs budget vs prior year, 4.2 monthly trend, 4.3 category table
5. Budget vs Actual — Root Cause (strategy gap table)
6. Balance Sheet & Working Capital (from ABP/strategy)
7. Risk Register
8. FY next-year plan & 5-year outlook
9. Business Case — the Ask (problem verdict, 5 plays, investment & enablers, board asks, success criteria)
10. Appendix — Source Files (paths in the Drive + DataMart tables)

Styling rules: negative values in red, positive in green; totals in a dark header row; numbers
right-aligned with tabular-nums; severity tags (Critical/High/Med/Low); one responsive page,
print-ready. BDT Crore = value / 1e7.

### STEP 8 — Verify & present

- Open the HTML file after writing to confirm it renders (optional, if a browser tool is available).
- Report to the user: file path, the three headline numbers (Revenue, GM%, PAT) for the FY, the
  decision asked of the board, and any data caveats (budget discrepancy, missing Drive doc, etc.).

## Reconciliation — completion criteria

The step is not done until:

- [ ] BU id + code resolved (or explicitly flagged unverifiable)
- [ ] Actuals pulled for target FY **and** prior FY; monthly and category queries succeeded
- [ ] Budget pulled for target FY; budget discrepancies flagged, not hidden
- [ ] Strategy/ABP docs found and mined (or documented as missing, with fallback used)
- [ ] P&L arithmetic reconciles (GP, EBITDA, PAT each derived from pulled lines)
- [ ] HTML written from the template with all 10 sections filled and all figures sourced

## Gotchas

- **Signs:** DataMart stores revenue/income negative, costs positive. Convert before presenting.
- **BU scope:** some strategy docs exclude sister BUs (AEL excluded Fariq Agro BU-189 and Hashem
  Rice Mills BU-188). Carry the same scope note into the doc.
- **Budget headers:** `fin.tblISbudget` may hold several `intBudgetHeaderId` per BU; group by
  header if numbers look duplicated. Confirm `strFiscalYear` matches the FY asked.
- **Big files:** delegate large doc extraction to explore subagents; keep your own context for
  synthesis, not for slurping 5 MB workbooks.
- **"Actual" labels in ABPs** are often YTD-May + June forecast, not audited full-year. Prefer the
  DataMart ERP sum for actuals; use ABP figures as the planning view and label them as such.
- **Missing columns:** partner/customer columns don't exist on `tblISTransaction` (no partner
  dimension); category analysis uses `intProfitCenterId`/`strProfitCenterName` instead.
