# RemAT Endpoint Compliance Dashboard — Build Guide

Deliverable: a `.pbix` matching `RemAT_Dashboard_Sketch.pptx`, fed from the VBA
workbook + the Summary of Releases file, with a scheduled refresh.

## Build order (do these in Power BI Desktop)

1. **Import the theme** — View → Themes → Browse for themes → `03_RemAT_Theme.json`.
   The canvas turns dark navy to match the sketch.
2. **Load data** — Home → Transform data → paste each query from `01_PowerQuery_M.md`
   (`PowerBI_Export`, `VIP_Current`, `Releases`, `Date`). Close & Apply.
3. **Model** — Model view:
   - Mark `Date` as the date table.
   - (Optional) relate `Date[Date]` → `PowerBI_Export[First_Seen_Date]`.
   - `Releases` stays standalone (different grain — it's per-bulletin, not per-device).
4. **Measures** — paste everything from `02_DAX_Measures.md`.
5. **Build the page** — see layout map below.
6. **Publish & refresh** — see last section.

## Page layout map (single page, matches the sketch)

| Sketch area | Visual to use | Field / measure |
|---|---|---|
| Top filter chips | Slicers (tile style) | `Operating_System`, `Is_VIP`, `Aging_Bucket` |
| 6 KPI cards | Card visuals | `Avg Compliance`, `Noncompliant Count`, `Avg Days`, `VIP Exposed`, + 2 agent-health (mock) |
| Compliance by Bulletin | Table / Matrix | `Releases`: Applications, Severity, Type, `Total Applicable`, `Avg Compliance` |
| Aging buckets | Clustered bar | axis `Aging_Bucket`, value `Noncompliant Count` |
| Actionable vs Unreachable | Donut (needs status col — mock for now) | — |
| Agent Health SMP/SEP | Gauge / cards (mock) | — |
| Failure categories | Bar (mock) | — |
| Trend over time | Line chart (needs history — mock) | — |
| Remediated vs New | Cards (needs history — mock) | — |
| Items Requiring Action | Table w/ conditional color | `Releases` filtered `Comp. % < 0.925`, color by `Compliance Color` |

✅ = real data today: KPI cards (compliance + quarantine), bulletin table, aging buckets, action items.
🟡 = mock/placeholder until client supplies data: agent health, failure categories, trend, remediated, actionable/unreachable.

## Publish & scheduled refresh

1. **Publish** — Home → Publish → pick a workspace in the Power BI Service.
2. **Data source for refresh:**
   - If the `.xlsm` stays on a local/network path → install **On-premises Data Gateway**
     and bind the dataset to it.
   - **Recommended:** have the client put the workbook in **SharePoint/OneDrive** →
     no gateway needed, and switch `Releases` to the SharePoint query (Query 3-PROD).
3. **Schedule** — Dataset → Settings → Scheduled refresh → set cadence to their patch
   cycle (confirm: weekly or monthly).

## Before you go far — confirm with client
- Source of agent-health / failure-category / trend / actionable data (the 🟡 tiles).
- Will production data use this same workbook layout (just larger)?
- Workbook location for refresh: SharePoint/OneDrive (preferred) or local (needs gateway)?
- SharePoint folder URL for the Releases file + whether it's re-dated each cycle.
- Refresh frequency.
