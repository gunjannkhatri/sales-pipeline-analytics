# Power Query (M) Code

Create a parameter called `BaseUrl` first (Home → Manage Parameters → New):

- **From GitHub (works for anyone opening the file):**
  `https://raw.githubusercontent.com/gunjannkhatri/sales-pipeline-analytics/main/data/`
- **From a local folder:** `C:\path\to\sales-pipeline-analytics\data\` (swap `Web.Contents` for `File.Contents` in the queries below)

---

## Fact_Leads

```m
let
    Source = Csv.Document(
        Web.Contents(BaseUrl & "fact_leads.csv"),
        [Delimiter = ",", Encoding = 65001, QuoteStyle = QuoteStyle.Csv]
    ),
    Promoted = Table.PromoteHeaders(Source, [PromoteAllScalars = true]),
    Typed = Table.TransformColumnTypes(Promoted, {
        {"LeadID", type text},
        {"CreatedDate", type date},
        {"SourceKey", Int64.Type},
        {"OwnerKey", Int64.Type},
        {"ProductKey", Int64.Type},
        {"CurrentStageKey", Int64.Type},
        {"MaxStageOrder", Int64.Type},
        {"Status", type text},
        {"LastActivityDate", type date},
        {"WonDate", type date},
        {"LostDate", type date},
        {"LostReason", type text},
        {"ExpectedValue", type number},
        {"Revenue", type number},
        {"CycleDays", Int64.Type}
    }, "en-US"),
    LostReasonClean = Table.ReplaceValue(Typed, "", null, Replacer.ReplaceValue, {"LostReason"})
in
    LostReasonClean
```

## Dimension tables (same pattern for each)

```m
// Dim_Source
let
    Source = Csv.Document(Web.Contents(BaseUrl & "dim_source.csv"), [Delimiter = ",", Encoding = 65001]),
    Promoted = Table.PromoteHeaders(Source, [PromoteAllScalars = true]),
    Typed = Table.TransformColumnTypes(Promoted, {{"SourceKey", Int64.Type}, {"Source", type text}, {"SourceGroup", type text}})
in
    Typed
```

```m
// Dim_Owner
let
    Source = Csv.Document(Web.Contents(BaseUrl & "dim_owner.csv"), [Delimiter = ",", Encoding = 65001]),
    Promoted = Table.PromoteHeaders(Source, [PromoteAllScalars = true]),
    Typed = Table.TransformColumnTypes(Promoted, {{"OwnerKey", Int64.Type}, {"SalesOwner", type text}})
in
    Typed
```

```m
// Dim_Product
let
    Source = Csv.Document(Web.Contents(BaseUrl & "dim_product.csv"), [Delimiter = ",", Encoding = 65001]),
    Promoted = Table.PromoteHeaders(Source, [PromoteAllScalars = true]),
    Typed = Table.TransformColumnTypes(Promoted, {{"ProductKey", Int64.Type}, {"Product", type text}})
in
    Typed
```

```m
// Dim_Stage
let
    Source = Csv.Document(Web.Contents(BaseUrl & "dim_stage.csv"), [Delimiter = ",", Encoding = 65001]),
    Promoted = Table.PromoteHeaders(Source, [PromoteAllScalars = true]),
    Typed = Table.TransformColumnTypes(Promoted, {{"StageKey", Int64.Type}, {"Stage", type text}, {"StageOrder", Int64.Type}, {"Probability", type number}}, "en-US")
in
    Typed
```

```m
// Dim_Date
let
    Source = Csv.Document(Web.Contents(BaseUrl & "dim_date.csv"), [Delimiter = ",", Encoding = 65001]),
    Promoted = Table.PromoteHeaders(Source, [PromoteAllScalars = true]),
    Typed = Table.TransformColumnTypes(Promoted, {
        {"Date", type date}, {"Year", Int64.Type}, {"MonthNum", Int64.Type}, {"Month", type text},
        {"YearMonth", type text}, {"MonthStart", type date}, {"WeekStart", type date},
        {"Weekday", type text}, {"IsWeekend", type logical}
    }, "en-US")
in
    Typed
```

```m
// Fact_Targets
let
    Source = Csv.Document(Web.Contents(BaseUrl & "fact_targets.csv"), [Delimiter = ",", Encoding = 65001]),
    Promoted = Table.PromoteHeaders(Source, [PromoteAllScalars = true]),
    Typed = Table.TransformColumnTypes(Promoted, {{"MonthStart", type date}, {"RevenueTarget", type number}, {"LeadTarget", Int64.Type}}, "en-US")
in
    Typed
```

---

## Notes

- Sort `Dim_Source[Source]` and `Dim_Stage[Stage]` using `StageOrder` (Column tools → Sort by column) so the funnel shows in the right order.
- In production, replace the CSV sources with your CRM export (LeadSquared / Zoho / HubSpot) via Power Automate → OneDrive/SharePoint or a SQL view. The model and DAX stay the same.
