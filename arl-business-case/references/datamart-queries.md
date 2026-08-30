# DataMart SQL References (validated against BI DataMart)

Server: `203.202.241.211` / `DataMart` (via `ARL-DataMart_sql_query`). Read-only. Use `dbo` unless noted.
All tables carry `intBusinessUnitId` (AEL = 144, ACCL = 4, etc. — resolve via STEP 1).

## 1. Resolve SBU

```sql
SELECT intBusinessUnitId, strBusinessUnitCode, strBusinessUnitName,
       strBusinessUnitGroupName, strBusinessUnitSubGroupname, numTaxRate
FROM dbo.tblBusinessUnit
WHERE strBusinessUnitName LIKE '%Akij Essentials%' OR strBusinessUnitCode = 'AEL'
```

## 2. Actual P&L by FS component (target & prior FY)

```sql
SELECT intFSComponentId, strFSComponentName, strType, SUM(numAmount) AS TotalAmount
FROM dbo.tblISTransaction
WHERE intBusinessUnitId = 144
  AND dteTransactionDate >= '2025-07-01' AND dteTransactionDate <= '2026-06-30'
GROUP BY intFSComponentId, strFSComponentName, strType
ORDER BY intFSComponentId
```

## 3. Monthly revenue / COGS / finance cost

```sql
SELECT YEAR(dteTransactionDate) Yr, MONTH(dteTransactionDate) Mn,
       SUM(CASE WHEN intFSComponentId=1  THEN -numAmount ELSE 0 END) Revenue,
       SUM(CASE WHEN intFSComponentId=20 THEN  numAmount ELSE 0 END) COGS,
       SUM(CASE WHEN intFSComponentId=6  THEN  numAmount ELSE 0 END) FinCost
FROM dbo.tblISTransaction
WHERE intBusinessUnitId=144 AND dteTransactionDate>='2025-07-01' AND dteTransactionDate<='2026-06-30'
GROUP BY YEAR(dteTransactionDate), MONTH(dteTransactionDate)
ORDER BY Yr, Mn
```

## 4. Profit-center (category) P&L

```sql
SELECT intProfitCenterId, strProfitCenterName, intFSComponentId, strFSComponentName, SUM(numAmount) Amt
FROM dbo.tblISTransaction
WHERE intBusinessUnitId=144 AND dteTransactionDate>='2025-07-01' AND dteTransactionDate<='2026-06-30'
  AND intFSComponentId IN (1,20,21)
GROUP BY intProfitCenterId, strProfitCenterName, intFSComponentId, strFSComponentName
ORDER BY intProfitCenterId, intFSComponentId
```

## 5. Budget by FS component

```sql
SELECT strFiscalYear, intFSComponentId, strFSComponentName, SUM(numAmount) BudgetAmount
FROM fin.tblISbudget
WHERE intBusinessUnitId=144
GROUP BY strFiscalYear, intFSComponentId, strFSComponentName
ORDER BY strFiscalYear, intFSComponentId
```

## FS component map (Income Statement)

| ID | FS Component | P&L line | Sign |
|---|---|---|---|
| 1 | Gross Sales Revenue | Revenue | − (credit) |
| 20 | Cost Of Goods Sold | COGS | + (debit) |
| 21 | Manufacturing Overhead | MOH | + |
| 3 | Administrative Expenses | OpEx | + |
| 4 | Marketing Expenses | OpEx | + |
| 22 | Selling Expenses | OpEx | + |
| 25 | Logistics & Distribution Expenses | OpEx | + |
| 5 | Other Operating Gain/Loss | Other | ± |
| 7 | Depreciation Expenses | Dep | + |
| 6 | Financial Expenses | Fin cost | + |
| 8 | Non Operating Income | Other income | − (credit) |
| 9 | Provision For Income Tax | Tax | + |
| 19 | Sales Tax (SD & VAT) | Tax | + |

## P&L arithmetic (reconcile check)

```
Revenue  = −FS1
COGS     =  FS20
MOH      =  FS21
GP       =  Revenue − COGS − MOH
OpEx     =  FS3 + FS4 + FS22 + FS25 (+FS5)
EBITDA   =  GP − OpEx
EBIT     =  EBITDA − FS7
PBT      =  EBIT − FS6 + (−FS8)
PAT      =  PBT − FS9 − FS19
```

## Known quirks

- `tblISTransaction` has **no partner/customer column** — category analysis is profit-center based.
- `fin.tblISbudget` may hold multiple `intBudgetHeaderId` per BU; group by header if totals look duplicated.
- Budget rows are monthly (12 rows per line); `SUM` across months = full-year budget.
- Some FS lines (e.g. FS21 MOH) may be absent from a BU's budget — that's normal, not an error.
- ABP workbooks' "FY actual" columns are often YTD-May + Jun forecast — label as planning view, prefer DataMart sum for actuals.
