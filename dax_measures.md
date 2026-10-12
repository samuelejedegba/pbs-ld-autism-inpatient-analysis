# DAX measures

Patterns used throughout the model. Table and column names match the Power Query output in `/data`.

## Pattern 1: helper + wrapper (latest month)
A measure cannot reference itself, so each "latest month" figure uses a hidden raw helper (`_X Raw`) and a wrapper that finds the last month with data.

```dax
Total Inpatients =
CALCULATE (
    SUM ( Fact_National_TimeSeries[Value] ),
    Fact_National_TimeSeries[Measure] = "All inpatients (at month end)",
    Fact_National_TimeSeries[Unit] = "Numbers",
    Fact_National_TimeSeries[Table] = "Table 1.1 Commissioner submission status and inpatient flow"
)

Total Inpatients (Latest) =
CALCULATE (
    [Total Inpatients],
    LASTNONBLANK ( Fact_National_TimeSeries[Date], [Total Inpatients] )
)
```

## Net flow
```dax
Net Flow (Latest Month) = [Admissions (Latest Month)] - [Discharges (Latest Month)]
```

## Destination group (calculated column)
Groups discharge outcomes into three buckets. Only exists in the data from March 2022.
```dax
Destination Group =
SWITCH (
    TRUE (),
    Fact_National_TimeSeries[Measure] = "Discharged to community settings", "Community",
    Fact_National_TimeSeries[Measure] = "Transferred to other hospital beds", "Other hospital",
    Fact_National_TimeSeries[Measure] IN { /* the seven other outcome labels, copied exactly from the data */ }, "Other",
    BLANK ()
)
```

## Long stays
`_Over 2 Years Raw` sums the bands "Two years up to five years", "Five years up to ten years" and "More than ten years" where Section = "Total length of stay2".

## Regional variation
```dax
Highest ICB Rate =
CALCULATE (
    MAXX ( VALUES ( Fact_Regional_Monthly[ICB_Code] ), [Rate Per Million (Latest Month)] ),
    Fact_Regional_Monthly[Org_Type] = "ICB"
)

Regional Autism Share = DIVIDE ( [_Regional Autism], [_Regional Total] )
```
The `Org_Type = "ICB"` filter keeps the TCP rows (e.g. Suffolk) out of the ICB charts.

## Restraint
```dax
% Change Since Oct 2024 =
DIVIDE (
    [Interventions (Latest Month)] - [_Oct 2024 Interventions],
    [_Oct 2024 Interventions]
)

Under 18 Interventions Per Person = DIVIDE ( [_Under 18 Raw], [_Under 18 People Raw] )

Highest Age Band =
CALCULATE (
    SELECTCOLUMNS (
        TOPN ( 1, VALUES ( Fact_Restraint_Age_Trend[Age_Band] ), [_Age Band Total], DESC ),
        Fact_Restraint_Age_Trend[Age_Band]
    )
)
```

## Lessons learned
- Copy label text exactly from the data. Truncated or retyped labels silently return blank.
- Chart measures should not hard-code the category. Let the visual axis slice them.
- `FORMAT()` returns text, so do not use it in measures that other measures reuse.
- Calculated columns (row context) and measures (filter context) are not interchangeable.
- Do not add up "people restrained" across months. The same person is counted every month they are restrained.
