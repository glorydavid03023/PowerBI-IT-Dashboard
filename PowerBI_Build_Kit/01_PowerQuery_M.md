# Power Query (M) — paste into Power BI Desktop

How to use: **Home → Transform data** (opens Power Query) → **New Source → Blank Query**
→ **View → Advanced Editor** → delete everything → paste one query below → Done.
Repeat for each query. Then **Close & Apply**.

---

## Query 1 — `PowerBI_Export`  (quarantine / endpoint table)

Primary fact table. Connects straight to the named table `tblPowerBIExport`.

> ⚠️ KNOWN SOURCE-FILE BUG (confirmed in the sample): the export's
> `Days_In_Quarantine` and `Aging_Bucket` columns are all `#VALUE!` errors
> (because `First_Seen_Date` is stored as text, so `TODAY()-text` fails), and
> `Is_VIP` is empty. The query below **recomputes all three** so the dashboard
> works regardless. Also flag the broken macro to the client.

```m
let
    Source = Excel.Workbook(
        File.Contents("f:\Power BI\Retest1_ Retest_Retest_ RemAT_Workbook_Improved_Freelance.xlsm"),
        null, true),
    tbl = Source{[Item="tblPowerBIExport", Kind="Table"]}[Data],

    // First_Seen_Date is text like "06/10/2026" -> parse to a real date (US format)
    asDate = Table.TransformColumnTypes(tbl, {{"First_Seen_Date", type date}}, "en-US"),

    // drop the broken #VALUE! columns AND the empty original Is_VIP (it gets recomputed below)
    dropped = Table.RemoveColumns(asDate, {"Days_In_Quarantine", "Aging_Bucket", "Is_VIP"}),

    // recompute Days_In_Quarantine = today - first seen
    withDays = Table.AddColumn(dropped, "Days_In_Quarantine",
        each Duration.Days(Date.From(DateTime.LocalNow()) - [First_Seen_Date]), Int64.Type),

    // recompute Aging_Bucket from the days
    withBucket = Table.AddColumn(withDays, "Aging_Bucket", each
        let d = [Days_In_Quarantine] in
        if d = null then null
        else if d <= 3 then "0-3 Days"
        else if d <= 7 then "4-7 Days"
        else if d <= 14 then "8-14 Days"
        else "15+ Days", type text),

    // recompute Is_VIP by matching Computer_Name against the VIP list
    joined = Table.NestedJoin(withBucket, {"Computer_Name"}, VIP_Current, {"Computer Name"}, "vip", JoinKind.LeftOuter),
    withVIP = Table.AddColumn(joined, "Is_VIP", each if Table.RowCount([vip]) > 0 then "Yes" else "No", type text),
    cleaned = Table.RemoveColumns(withVIP, {"vip"})
in
    cleaned
```

---

## Query 2 — `VIP_Current`  (optional detail table)

```m
let
    Source = Excel.Workbook(
        File.Contents("f:\Power BI\Retest1_ Retest_Retest_ RemAT_Workbook_Improved_Freelance.xlsm"),
        null, true),
    tbl = Source{[Item="tblVIPCurrent", Kind="Table"]}[Data]
in
    tbl
```

---

## Query 3 — `Releases`  (patch compliance — DEV version, local file)

Use this while building. The sheet has a banner row + two stacked blocks
(software releases, then Microsoft security patches), so we keep only rows that
have a real Compliance % number.

```m
let
    Source = Excel.Workbook(
        File.Contents("f:\Power BI\Summary of Releases 061526.xlsx"),
        null, true),
    sheet = Source{[Item="Summary of Releases", Kind="Sheet"]}[Data],
    promoted = Table.PromoteHeaders(Table.Skip(sheet, 1), [PromoteAllScalars=true]),
    // keep only rows where the compliance column is a number
    cleaned = Table.SelectRows(promoted, each [#"Comp. %"] <> null and [#"Comp. %"] <> "Comp. %"),
    typed = Table.TransformColumnTypes(cleaned, {
        {"Applications", type text},
        {"Type", type text},
        {"Severity", type text},
        {"Released", type date},
        {"Applicable", Int64.Type},
        {"Vulnerable", Int64.Type},
        {"Installed", Int64.Type},
        {"Comp. %", Percentage.Type}
    })
in
    typed
```

> The Releases sheet is messy by hand-entry. After loading, open the table and
> verify column names match (the header text has hidden line breaks). Adjust the
> `TransformColumnTypes` names to whatever Power Query actually shows.

---

## Query 3-PROD — `Releases` from SharePoint folder (use this for the live refresh)

Swap Query 3 for this once the client confirms the SharePoint folder URL. It auto-picks
the newest file (the `MMDDYY` date in the name sorts newest-last; we grab most-recent by date).

```m
let
    Source = SharePoint.Files("https://YOURTENANT.sharepoint.com/sites/YOURSITE", [ApiVersion = 15]),
    inFolder = Table.SelectRows(Source, each Text.Contains([Folder Path], "Summary of Releases")
                  and Text.EndsWith([Name], ".xlsx")),
    newest = Table.FirstN(Table.Sort(inFolder, {{"Date modified", Order.Descending}}), 1),
    content = newest{0}[Content],
    book = Excel.Workbook(content, null, true),
    sheet = book{[Item="Summary of Releases", Kind="Sheet"]}[Data],
    promoted = Table.PromoteHeaders(Table.Skip(sheet, 1), [PromoteAllScalars=true]),
    cleaned = Table.SelectRows(promoted, each [#"Comp. %"] <> null and [#"Comp. %"] <> "Comp. %")
in
    cleaned
```

---

## Query 4 — `Date` table (for trend + time intelligence)

```m
let
    StartDate = #date(2026, 1, 1),
    EndDate   = #date(2026, 12, 31),
    Dates = List.Dates(StartDate, Duration.Days(EndDate - StartDate) + 1, #duration(1,0,0,0)),
    T = Table.FromList(Dates, Splitter.SplitByNothing(), {"Date"}),
    typed = Table.TransformColumnTypes(T, {{"Date", type date}}),
    withCols = Table.AddColumn(typed, "Year", each Date.Year([Date]), Int64.Type),
    withMonthNo = Table.AddColumn(withCols, "MonthNo", each Date.Month([Date]), Int64.Type),
    withMonth = Table.AddColumn(withMonthNo, "Month", each Date.ToText([Date], "MMM"), type text)
in
    withMonth
```
After load: **Modeling → Mark as date table → Date**.
