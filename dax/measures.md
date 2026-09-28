# DAX Measures

All measures live in a dedicated table called `_Measures` (Enter Data → blank table → move measures into it).

**Model setup first**
- Mark `Dim_Date` as a date table (`Date` column).
- `Fact_Leads` → `Dim_Date` has **three** relationships:
  - `CreatedDate` → `Dim_Date[Date]` — **active** (lead cohort view)
  - `WonDate` → `Dim_Date[Date]` — inactive (revenue / deals by close date)
  - `LostDate` → `Dim_Date[Date]` — inactive (losses by close date)
- `Fact_Targets[MonthStart]` → `Dim_Date[Date]` (targets sit on the 1st of each month, so use month-level visuals).
- `Dim_Stage` → `Fact_Leads[CurrentStageKey]` (used for open pipeline by stage).

---

## 1. Volume

```dax
Total Leads = COUNTROWS ( Fact_Leads )
```

```dax
Qualified Leads =
CALCULATE ( [Total Leads], Fact_Leads[MaxStageOrder] >= 3 )
```

```dax
Qualification Rate = DIVIDE ( [Qualified Leads], [Total Leads] )
```

## 2. Outcomes (lead cohort — by the date the lead was created)

```dax
Won Deals = CALCULATE ( [Total Leads], Fact_Leads[Status] = "Won" )
```

```dax
Lost Deals = CALCULATE ( [Total Leads], Fact_Leads[Status] = "Lost" )
```

```dax
Open Deals = CALCULATE ( [Total Leads], Fact_Leads[Status] = "Open" )
```

```dax
-- Won ÷ (Won + Lost). Open deals are left out so recent leads don't drag the rate down.
Win Rate = DIVIDE ( [Won Deals], [Won Deals] + [Lost Deals] )
```

```dax
Lead to Won % = DIVIDE ( [Won Deals], [Total Leads] )
```

## 3. Revenue (by close date)

```dax
Revenue =
CALCULATE (
    SUM ( Fact_Leads[Revenue] ),
    USERELATIONSHIP ( Dim_Date[Date], Fact_Leads[WonDate] )
)
```

```dax
Deals Closed Won =
CALCULATE (
    COUNTROWS ( Fact_Leads ),
    Fact_Leads[Status] = "Won",
    USERELATIONSHIP ( Dim_Date[Date], Fact_Leads[WonDate] )
)
```

```dax
Avg Deal Size = DIVIDE ( [Revenue], [Deals Closed Won] )
```

```dax
Avg Sales Cycle (Days) =
CALCULATE (
    AVERAGE ( Fact_Leads[CycleDays] ),
    USERELATIONSHIP ( Dim_Date[Date], Fact_Leads[WonDate] )
)
```

```dax
Revenue MoM % =
VAR Prev =
    CALCULATE ( [Revenue], DATEADD ( Dim_Date[Date], -1, MONTH ) )
RETURN
    DIVIDE ( [Revenue] - Prev, Prev )
```

## 4. Targets

```dax
Revenue Target = SUM ( Fact_Targets[RevenueTarget] )
```

```dax
Lead Target = SUM ( Fact_Targets[LeadTarget] )
```

```dax
Revenue vs Target % = DIVIDE ( [Revenue], [Revenue Target] )
```

```dax
Lead Target Attainment % = DIVIDE ( [Total Leads], [Lead Target] )
```

## 5. Funnel

`Dim_Stage` rows with `StageOrder` 1–6 are the funnel steps. Filter the funnel visual to `StageOrder <= 6` (removes "Lost").

```dax
-- Leads that reached AT LEAST this stage
Funnel Leads =
VAR StepOrder = MAX ( Dim_Stage[StageOrder] )
RETURN
    CALCULATE (
        COUNTROWS ( Fact_Leads ),
        REMOVEFILTERS ( Dim_Stage ),
        Fact_Leads[MaxStageOrder] >= StepOrder
    )
```

```dax
-- % of leads that moved on from the previous step
Step Conversion % =
VAR StepOrder = MAX ( Dim_Stage[StageOrder] )
VAR PrevStep =
    CALCULATE (
        COUNTROWS ( Fact_Leads ),
        REMOVEFILTERS ( Dim_Stage ),
        Fact_Leads[MaxStageOrder] >= StepOrder - 1
    )
RETURN
    IF ( StepOrder = 1, BLANK (), DIVIDE ( [Funnel Leads], PrevStep ) )
```

## 6. Open pipeline health

```dax
-- Use TODAY() with live data. Fixed date keeps the sample dashboard stable.
Report Date = DATE ( 2026, 9, 28 )
```

```dax
Open Pipeline Value =
CALCULATE ( SUM ( Fact_Leads[ExpectedValue] ), Fact_Leads[Status] = "Open" )
```

```dax
-- Expected value × stage probability (from Dim_Stage)
Weighted Pipeline =
SUMX (
    FILTER ( Fact_Leads, Fact_Leads[Status] = "Open" ),
    Fact_Leads[ExpectedValue] * RELATED ( Dim_Stage[Probability] )
)
```

```dax
-- Open deals with no activity for 14+ days
Stalled Deals =
COUNTROWS (
    FILTER (
        Fact_Leads,
        Fact_Leads[Status] = "Open"
            && DATEDIFF ( Fact_Leads[LastActivityDate], [Report Date], DAY ) > 14
    )
)
```

```dax
Stalled Deals % = DIVIDE ( [Stalled Deals], [Open Deals] )
```

```dax
Avg Open Deal Age (Days) =
AVERAGEX (
    FILTER ( Fact_Leads, Fact_Leads[Status] = "Open" ),
    DATEDIFF ( Fact_Leads[CreatedDate], [Report Date], DAY )
)
```

## 7. Lost analysis

```dax
Lost Revenue Potential =
CALCULATE ( SUM ( Fact_Leads[ExpectedValue] ), Fact_Leads[Status] = "Lost" )
```

```dax
-- Use with LostReason on the axis
Lost Share % =
DIVIDE ( [Lost Deals], CALCULATE ( [Lost Deals], REMOVEFILTERS ( Fact_Leads[LostReason] ) ) )
```

## 8. Dynamic titles (optional)

```dax
Selected Period Label =
VAR MinD = MIN ( Dim_Date[Date] )
VAR MaxD = MAX ( Dim_Date[Date] )
RETURN FORMAT ( MinD, "DD MMM YYYY" ) & " – " & FORMAT ( MaxD, "DD MMM YYYY" )
```
