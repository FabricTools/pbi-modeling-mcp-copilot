# Sales Model — Improvement Suggestions

> Model: **Sales** | Workspace: **Git Demo**  
> Reviewed: 2026-04-24  
> Tool: pbi-modeling-mcp-copilot

---

## Table Overview (Row Counts)

| Table | Rows | Description |
|---|---|---|
| About | 4 | Key/Value metadata about the model |
| Calendar | 1,461 | Date spine (~4 years) |
| Customer | 5,585 | Customer master data |
| Dynamic Measure | 5 | Field-parameter-style measure switcher |
| Parameter - Dimension | 3 | Field parameter for dimension switching |
| Parameter - Measure | 6 | Field parameter for measure switching |
| Product | 2,517 | Product catalog |
| Sales | 2,881 | Fact table (transactions) |
| Smart Calcs | 5 | Calculation group items |
| Store | 74 | Store metadata |

---

## Identified Issues and Suggestions

### 1. Typo — Double Space in Measure Names

**Severity:** Low  
**Affected measures:**
- `Sales Amount  (Δ LY)` — has two spaces before `(Δ LY)`
- `Sales Amount  (% Δ LY)` — has two spaces before `(% Δ LY)`

**Suggestion:** Rename both measures to use a single space: `Sales Amount (Δ LY)` and `Sales Amount (% Δ LY)`. Note that any DAX referencing these measures must also be updated (e.g., the `Value` dynamic measure in the `Dynamic Measure` table references `[Sales Amount  (% Δ LY)]`).

---

### 2. Inconsistent Format Strings for Currency Measures

**Severity:** Medium  
**Affected measures:**

| Measure | Format String |
|---|---|
| `Sales Amount` | `$ #,##0` |
| `Sales Amount (LY)` | `\$#,0;(\$#,0);\$#,0` |
| `Sales Amount (YTD)` | `$ #,##0` |
| `Sales Amount (YTD, LY)` | `$ #,##0` |
| `Sales Amount (Δ LY)` | `$ #,##0` |
| `Cost` | `$ #,##0` |
| `Margin` | `$ #,##0` |
| `Margin (ly)` | `$ #,##0` |
| `Sales Amount (12M average)` | `$ #,##0` |
| `Sales Amount (6M average)` | `$ #,##0` |

`Sales Amount (LY)` uses a different format string pattern (`\$#,0;(\$#,0);\$#,0`) with no thousands-separator thousands and a parenthesis for negatives. This is inconsistent with the rest of the currency measures.

**Suggestion:** Standardise all currency measures to `$ #,##0`.

---

### 3. Missing Descriptions on Most Measures

**Severity:** Medium  
**Measures with no description (most of them):**

`Cost`, `Margin`, `Margin %`, `Margin % Overall`, `Margin (ly)`, `# Sales`, `Sales Qty`, `Sales Qty by Delivery Date`, `Sales Amount (YTD)`, `Sales Amount (YTD, LY)`, `Sales Amount (Δ LY)`, `Sales Amount (% Δ LY)`, `Sales Amount Avg per Day`, `Sales Amount (6M average)`, `# Customers`, `# Customers (with Sales)`, `# Products`, `# Products (with Sales)`, `# Stores`, `Value (ly)`, `Value (ytd)`, `Value Avg per Month`, `Value Daily Max`, `Value % (Δ ly)`, `Value Normalized (by date)`

**Suggestion:** Add a short plain-English description to every measure explaining what it calculates and any important caveats (e.g., which date relationship is used, what period is covered).

---

### 4. No Display Folders on Measures

**Severity:** Medium  

None of the 29 measures are organised into display folders, so the field list in report tools shows a flat, hard-to-navigate list.

**Suggestion:** Group measures into folders such as:
- `Sales / Amount`
- `Sales / Quantity`
- `Sales / Margin`
- `Counts`
- `Time Intelligence`
- `Moving Averages`
- `Dynamic Measure` (already isolated in its own table — keep as-is)

---

### 5. `Sales Qty` Uses `SUM` While All Other Additive Measures Use `SUMX`

**Severity:** Low  
**Measure:** `Sales Qty`  
**Expression:** `sum('Sales'[Quantity])`

All other additive fact measures (`Sales Amount`, `Cost`, `Margin`) use `SUMX` iterating over the `Sales` table. While `SUM` is functionally equivalent here, the inconsistency is confusing and `SUM` uses implicit lowercase function casing.

**Suggestion:** Rewrite as `SUM ( 'Sales'[Quantity] )` (consistent casing and spacing) or `SUMX ( Sales, Sales[Quantity] )` to match the pattern of the other measures.

---

### 6. Commented-Out Code in `Value Normalized (by date)`

**Severity:** Low  
**Measure:** `Dynamic Measure.Value Normalized (by date)`

The measure contains two commented-out alternative implementations:
```dax
//VAR MinOfGroup = MINX(ALLSELECTED('Calendar'[Month (Year)], 'Calendar'[MonthYearId]), [Value])
//VAR MaxOfGroup = MAXX(ALLSELECTED('Calendar'[Month (Year)], 'Calendar'[MonthYearId]), [Value])
```

**Suggestion:** Remove dead/commented-out code. If these alternatives are needed for reference, document them in the measure description or a wiki page instead.

---

### 7. `Margin % Overall` Uses `ROUND` Instead of a Format String

**Severity:** Low  
**Measure:** `Sales.Margin % Overall`  
**Expression:**
```dax
ROUND ( CALCULATE( [Margin %], REMOVEFILTERS () ), 2 )
```

Rounding inside the DAX expression bakes the precision into the value itself, making it impossible for report authors to show more decimal places if needed. Rounding should be a presentation concern handled by the format string.

**Suggestion:** Remove `ROUND()` and set the format string to `#,##0.00 %` (same as `Margin %`) — or use a dedicated format string for this measure if a different precision is intentional.

---

### 8. Inconsistent Percentage Format Strings

**Severity:** Low  

| Measure | Format String |
|---|---|
| `Margin %` | `#,##0.00 %` |
| `Sales Amount  (% Δ LY)` | `#,##0.00 %` |
| `Value % (Δ ly)` | `0.00%;-0.00%;0.00%` |
| `Margin % Overall` | `#,##0.00 %` |

`Value % (Δ ly)` uses a three-part semicolon format while the others use a simpler single-part format.

**Suggestion:** Standardise all percentage measures to the same format string (e.g., `#,##0.00 %`).

---

### 9. `Dynamic Measure.Value` Uses Opaque Numeric SWITCH Codes

**Severity:** Medium  
**Measure:** `Dynamic Measure.Value`  
**Expression (excerpt):**
```dax
SWITCH (
    measureCode
    ,1, [Sales Amount]
    ,2, [Sales Amount  (% Δ LY)]
    ,3, [# Customers (with Sales)]
    ,4, [Sales Qty]
    ,5, [Margin]
    ,BLANK ()
)
```

The numeric codes `1`–`5` map to the `Dynamic Measure[Code]` column but the mapping is not obvious from the expression alone. If new measures are added, the SWITCH and the underlying data must be kept manually in sync.

**Suggestion:** Switch on `SELECTEDVALUE('Dynamic Measure'[Measure])` (the label string) instead of the numeric code, making the DAX self-documenting and removing the need to maintain a separate numeric mapping table.

---

### 10. Calendar Table Not Marked as Date Table

**Severity:** Medium  

The `Calendar` table is used for all time-intelligence calculations but it is not formally marked as a Date Table in the model. Without this marking, Power BI's automatic date tables may remain active (the annotation `__PBI_TimeIntelligenceEnabled` = `0` suppresses this, but it is not the same as officially marking the calendar table).

**Suggestion:** Mark `Calendar` as a Date Table using the `Date` column as the key. This makes time-intelligence behaviour explicit and removes ambiguity.

---

### 11. `forceUniqueNames` Is Disabled

**Severity:** Low  

The model has `forceUniqueNames: false`. When disabled, measure and column names can clash across tables, which can lead to ambiguous references in DAX and MDX clients.

**Suggestion:** Enable `forceUniqueNames: true` unless there is a specific compatibility reason not to.

---

### 12. Row-Level Security Roles Are Very Coarse

**Severity:** Medium  
**Roles:**
- `Stores Cluster 1` — filters `Store` to codes `{1, 2, 4}`
- `Stores Cluster 2` — filters `Store` to codes `{10, 11, 15, 8}`

Both roles contain only static hard-coded store codes. If stores are added or reassigned, the role definitions must be updated manually. Additionally, there is no customer-level or product-level security.

**Suggestion:**
- Replace the static `IN {}` filter with a dynamic lookup from a security mapping table (e.g., `Store[Cluster] = USERPRINCIPALNAME()` pattern) so that access adapts automatically as the data changes.
- Consider whether customer or product-level restrictions are needed.

---

### 13. `Sales[Environment]` Column — Unclear Purpose

**Severity:** Low  

The `Sales` fact table contains an `Environment` column (type `String`). Its purpose is not documented and it does not appear in any measure logic or relationship.

**Suggestion:** Add a column description explaining what `Environment` represents (e.g., production vs. test data marker). If it is only used for ETL filtering, consider hiding it from report authors.

---

### 14. `Sales[Time]` Column — Unclear Purpose

**Severity:** Low  

The `Sales` fact table has a `DateTime` column named `Time` in addition to `Order Date` and `Delivery Date`. Its purpose is not described anywhere.

**Suggestion:** Add a column description or, if unused, remove or hide the column.

---

### 15. `Customer[Age]` Column — Potentially Stale

**Severity:** Medium  

The `Customer` table exposes an `Age` (Int64) column. If this is calculated at ETL time rather than as a DAX calculated column based on `Birthday`, the values will become stale between refreshes.

**Suggestion:** Either replace `Age` with a calculated column `= DATEDIFF(Customer[Birthday], TODAY(), YEAR)` so it is always current, or document that `Age` is a snapshot taken at a specific reference date.

---

### 16. Missing Description on `Sales Amount (6M average)` (Unlike the 12M Version)

**Severity:** Low  

`Sales Amount (12M average)` has the description: *"12 Month moving average sales calculation"*  
`Sales Amount (6M average)` has no description.

**Suggestion:** Add *"6 Month moving average sales calculation"* as the description.

---

### 17. `About` Table Has No Refresh Logic Documented

**Severity:** Low  

The `About` table stores key/value metadata (4 rows). It is not clear whether this table is refreshed as part of the normal dataset refresh or whether its values (e.g., `last refresh`) are updated automatically.

**Suggestion:** Document the source and refresh mechanism for this table, especially the `last refresh` key.

---

### 18. No Model-Level Description

**Severity:** Low  

The model has no top-level description set.

**Suggestion:** Add a model description summarising its purpose, data sources, target audience, and refresh cadence.

---

## Summary Priority Table

| # | Issue | Severity |
|---|---|---|
| 1 | Double space in measure names | Low |
| 2 | Inconsistent format strings — currency | Medium |
| 3 | Missing descriptions on most measures | Medium |
| 4 | No display folders on measures | Medium |
| 5 | `Sales Qty` uses `SUM` inconsistently | Low |
| 6 | Commented-out code in measure | Low |
| 7 | `ROUND()` in `Margin % Overall` bakes in precision | Low |
| 8 | Inconsistent percentage format strings | Low |
| 9 | Dynamic Measure SWITCH uses opaque codes | Medium |
| 10 | Calendar not marked as Date Table | Medium |
| 11 | `forceUniqueNames` disabled | Low |
| 12 | RLS roles use static hard-coded store codes | Medium |
| 13 | `Sales[Environment]` undocumented | Low |
| 14 | `Sales[Time]` column undocumented | Low |
| 15 | `Customer[Age]` may become stale | Medium |
| 16 | `Sales Amount (6M average)` missing description | Low |
| 17 | `About` table refresh not documented | Low |
| 18 | No model-level description | Low |
