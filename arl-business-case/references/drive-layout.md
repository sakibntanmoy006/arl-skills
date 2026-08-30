# Strategy MCP Drive Layout (validated)

Drive root: **"AR- 05 Years Strategy 27-31"** = `1bno3WJa452ehSL1Hu1764AjmX7vBdkGe`

| Folder | ID | Contents |
|---|---|---|
| AR Strategy 05 Years (FY 26-27 to FY 30-31) | `178VoXrIeLJbiSgJGxcbsYxoY4YJNIuwV` | Per-SBU 5-year strategy folders |
| Business Plan & Budget | `1UeX3KNjHWfZwD7mWn7bwLiZYJnusNlPc` | Strategy & Budget 2026-27 |
| ├ Strategy & Budget 2026-27 | `1qB10F-bKjPMc4wj9d1LtVd4How-l-soJ` | — |
| │ ├ All SBU | `1B1ISOupgAP3CL1QBC8XSTqy11-ez8hlc` | Per-SBU ABP / budget files (xlsx, PDF, CFO packs) |
| │ └ Budget File | `1uUlux9huAleOUYFSGkCzOYuYeU0XRpOK` | Projected IS (all-SBU consolidated) |
| Coporate Strategy | `1KkmgaWtbolTcYVGB9c3aDRbcxcmeaId9` | Corporate plan, heatmap, strategic map, time-phased strategy |

## How to find an SBU's strategy folder

1. `ARL-Strategy_drive_folder_tree(folderId: "178VoXrIeLJbiSgJGxcbsYxoY4YJNIuwV", depth: 1, includeFiles: false)`
   → find the folder whose name matches the SBU (code or name).
2. `ARL-Strategy_drive_folder_tree` on that SBU folder (depth 1–2, includeFiles true) → list its files.
3. For the ABP/budget: `ARL-Strategy_drive_folder_tree(folderId: "1B1ISOupgAP3CL1QBC8XSTqy11-ez8hlc", depth: 1)`
   → SBU folder → list files.

## File preference order inside an SBU folder

1. `*_5Year_Strategy*.xlsx` / `*.html` (strategy narrative + 5-yr table)
2. `*ABP*.xlsx` (annual business plan: category P&L, volumes, capex, WC, sensitivities)
3. `*Income statement*.pdf` (CFO-certified IS)
4. `*Business Plan*.pdf` / `*Budget*.pdf` (master pack — often large, use a subagent)

## Known SBU folder examples

| SBU | Code | Strategy folder | Key files |
|---|---|---|---|
| Akij Essentials | AEL | `15bTJYat5CcperqNN9w3QGNXG7LZ61ovo` | AEL 5-Year Strategy html/xlsx; ABP 2026-27 xlsx (5 MB); IS 2026-31 pdf |
| Akij Cement | ACCL | `1L_ABWy7RW3Wf8coG9eC6BNSXzCCdqelT` | ACCL 5-Year Strategy html/xlsm |
| Akij Agro Feed | AAFL | `1ZI_57dl90P5W9dgndFBqp-mbvNzVxa5w` | AAFL 5-Year Strategy xlsx/html |
| AEL (Trading) | — | `1yjsUEqQs6YHAup_SS4Om-YQxS83dSJ7U` | AEL Trading 5-Year Strategy xlsm/html |
| AEFL (Electrofab) | AEFL | `1Hmp8s6DG9PQtVEm1HGb5E83eCDajANS5` | AEFL 5Y Strategy html/xlsx |

## Drive read notes

- `drive_read_file` parses xlsx/csv/pdf/docx/txt/html/google-native. Large or binary-heavy files
  may fail inline (unsupported binary, timeout, POST error). When it fails or is >40KB, hand the
  extraction to an **explore subagent** (it can download via public Drive export URLs and parse
  with `openpyxl` / `pdftotext`), asking it to return exact numbers only.
- Always record the Drive file id / path for the Appendix (section 10).
