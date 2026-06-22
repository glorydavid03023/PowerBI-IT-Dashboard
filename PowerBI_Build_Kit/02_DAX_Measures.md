# DAX Measures — paste into Power BI Desktop

How to use: right-click `PowerBI_Export` in the Data pane → **New measure** →
paste one block → Enter. Repeat. Put them all in `PowerBI_Export` (or make a
blank "_Measures" table to hold them tidily).

> `Is_VIP` handling: these assume `Is_VIP` is text "Yes"/"TRUE". If it's a logical
> column, replace `= "Yes"` with `= TRUE()`.

---

## Endpoint / quarantine KPIs (from `PowerBI_Export`)

```dax
Noncompliant Count = COUNTROWS ( PowerBI_Export )
```

```dax
Avg Days = AVERAGE ( PowerBI_Export[Days_In_Quarantine] )
```

```dax
Median Days = MEDIAN ( PowerBI_Export[Days_In_Quarantine] )
```

```dax
VIP Exposed =
CALCULATE (
    COUNTROWS ( PowerBI_Export ),
    PowerBI_Export[Is_VIP] = "Yes"
)
```

### Aging buckets (use these for the 0-3 / 4-7 / 8+ tile)

```dax
Aging 0-3 = CALCULATE ( [Noncompliant Count], PowerBI_Export[Aging_Bucket] = "0-3 Days" )
```

```dax
Aging 4-7 = CALCULATE ( [Noncompliant Count], PowerBI_Export[Aging_Bucket] = "4-7 Days" )
```

```dax
Aging 8+ =
CALCULATE (
    [Noncompliant Count],
    PowerBI_Export[Aging_Bucket] IN { "8-14 Days", "15+ Days" }
)
```

> Tip: a clustered bar with `Aging_Bucket` on the axis and `Noncompliant Count` as
> the value reproduces the sketch directly — no separate measures needed. The three
> measures above are for KPI cards.

---

## Patch compliance KPIs (from `Releases`)

Create these on the `Releases` table.

```dax
Avg Compliance = AVERAGE ( Releases[Comp. %] )
```

```dax
Total Applicable = SUM ( Releases[Applicable] )
```

```dax
Total Vulnerable = SUM ( Releases[Vulnerable] )
```

```dax
Bulletins Below Threshold =
CALCULATE (
    COUNTROWS ( Releases ),
    Releases[Comp. %] < 0.925
)
```

```dax
Compliance Color =
SWITCH (
    TRUE (),
    [Avg Compliance] >= 0.95, "#4ADE80",   -- green
    [Avg Compliance] >= 0.90, "#F59E0B",   -- amber
    "#EF4444"                               -- red
)
```
Use `Compliance Color` under a visual's **Conditional formatting → Font/Background → Field value**
to drive the green/amber/red of the bulletin table, matching the sketch.

---

## Notes on the ❌ tiles (no data yet)

Agent Health (SMP/SEP), Failure Categories, Trend over time, and Remediated-vs-New
have **no source columns** in the current files. Until the client provides them, either:
- leave those visuals out, or
- build a small manual "Mock" table (Home → Enter Data) with the sketch's example
  numbers so the page looks complete for the pilot — clearly label it "ILLUSTRATIVE".
