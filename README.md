# Time-to-Fill & Recruiting Funnel Leakage

A Power BI Project (PBIP) for recruiting analytics that goes past *"how long does
hiring take"* and answers the three questions a TA lead actually needs:

1. **How long** — and which of the three clocks is the slow one
2. **Where it leaks** — which funnel stage destroys the most candidates
3. **Who caused it** — candidate dropout, selection working as intended, or our own process

Built as a TMDL semantic model plus a 5-page PBIR report, with generated sample
data so the whole thing runs without connecting to a real ATS.

---

## Why this exists

Most recruiting dashboards report one number: average time to fill. That number
hides more than it reveals.

In the sample data it reads **73.6 days**. The median is **61.5** and P90 is
**140.4** — one requisition in ten takes more than four months. Promise a hiring
manager "about 74 days" and you will be wrong in both directions, often.

It also says nothing about the **7,091 applications that produced 133 hires**.
Time-to-fill cannot tell you where the other 6,958 went. This model can: the
largest controllable leak sits at **Hiring Manager Review**, which loses **863
candidates (54.9%)** while running **7.7 days against a 4-day SLA**. Slow and
lossy at the same time.

---

## Screens

| Page | Question it answers |
|---|---|
| Executive Summary | How long, how leaky, and where to click next |
| Time to Fill | Which clock is slow — approval, search, or offer-to-start |
| Funnel Leakage | Which stage destroys the most candidates, and who owns that loss |
| Source & Channel | Which channels deliver hires rather than applications |
| Pipeline Health | Which open roles are aging out, and is there pipeline behind them |

Page size 1920 × 1080, `FitToPage`. Dark theme registered at
`StaticResources/RegisteredResources/FunnelDark.json` — edit that one file to
reskin all five pages.

---

## The data model

Star schema, three facts at three different grains.

```
                     DimDate
                        |
    DimJob ---- FactRequisition ---- DimRecruiter
                        |
                 FactApplication ---- DimSource
                        |          \
                 FactStageEvent      DimRejectReason
                        |
                    DimStage
```

| Table | Grain | Rows |
|---|---|---|
| `FactRequisition` | one open role | 160 |
| `FactApplication` | one candidate application | 7,091 |
| `FactStageEvent` | one candidate entering one stage | 13,382 |
| `DimDate` | day (2024-01-01 → 2026-06-30) | 912 |
| `DimStage` | funnel stage, with `StageOrder` and `SLADays` | 8 |
| `DimSource` | channel, with a benchmark yield rate | 8 |
| `DimJob` / `DimRecruiter` / `DimRejectReason` | lookups | 20 / 7 / 11 |

**`FactStageEvent` is the table that makes this work.** Without one row per
candidate-per-stage you can count applications and hires and nothing in between —
which means leakage is not measurable at all. If your ATS only exports
applications and hires, that is the extract to go and ask for.

### Relationship design

`DimDate` is active only to `FactRequisition[PostedDateKey]`. Three other
relationships are **deliberately inactive** so the model has no ambiguous filter
paths:

| Inactive relationship | Reached via |
|---|---|
| `FactApplication[AppliedDateKey]` → `DimDate` | `USERELATIONSHIP` in `[Applications by Applied Date]` |
| `FactStageEvent[EnterDateKey]` → `DimDate` | available for stage-entry trending |
| `FactApplication[FurthestStageID]` → `DimStage` | keeps the stage slicer driving `FactStageEvent` |

---

## Measures

50 measures in a dedicated `_Measures` table, grouped by display folder.

| Folder | Covers |
|---|---|
| `01 Volume` | requisition, application and hire counts |
| `02 Time to Fill` | approval lag, TTF average / median / P90, time to start, SLA |
| `03 Funnel` | reach, advance, pass-through, leakage, conversion, offer acceptance |
| `04 Stage Velocity` | days in stage vs SLA, share of total cycle time |
| `05 Dropout & Quality` | withdrawals, company-caused losses, source yield |
| `06 Pipeline Health` | open req aging, active candidates, coverage ratio |
| `07 Conditional Formatting` | colour and label measures for cards and tables |

### The three that carry the report

Leakage, per stage:

```dax
Candidates Reaching Stage = DISTINCTCOUNT ( FactStageEvent[ApplicationID] )

Candidates Advanced =
    CALCULATE (
        [Candidates Reaching Stage],
        FactStageEvent[Outcome] IN { "Advanced", "Hired" }
    )

Leakage Rate = DIVIDE ( [Candidates Reaching Stage] - [Candidates Advanced],
                        [Candidates Reaching Stage] )
```

Survival against the top of the funnel rather than the previous stage:

```dax
Conversion from Applied =
VAR AppliedCount =
    CALCULATE (
        [Candidates Reaching Stage],
        ALL ( DimStage ),
        DimStage[StageOrder] = 1
    )
RETURN
    DIVIDE ( [Candidates Reaching Stage], AppliedCount )
```

The headline that names its own answer:

```dax
Worst Leak Stage =
VAR StageLeak =
    ADDCOLUMNS (
        FILTER ( ALL ( DimStage ), DimStage[StageOrder] < 8 ),
        "@Lost", [Leakage Count]
    )
VAR Ranked = TOPN ( 1, FILTER ( StageLeak, [@Lost] > 0 ), [@Lost], DESC )
RETURN
    CONCATENATEX ( Ranked, DimStage[StageName] )
```

`Worst Leak Stage` ranks by **count, not rate** — on purpose. Final Interview
leaks 43.5% but only 135 people; fixing it moves nothing. Recruiter Screen and HM
Review lose 2,184 between them. If you change this to rank by percentage, the
card will start contradicting the chart next to it.

### Calculated columns

On `FactRequisition`: `Days to Fill`, `Req Age Days`, `Aging Bucket`,
`Aging Bucket Order`, `SLA Status`.

`Aging Bucket` and `Aging Bucket Order` both derive from `Req Age Days`
independently. They look like they should chain — order translating the bucket
label into a number — but `Aging Bucket` is sorted by `Aging Bucket Order`, and
sort-by is a real dependency in the model graph. Chaining them is a circular
dependency. The cost of keeping them separate is that the thresholds are
duplicated; change the buckets and you must edit both.

---

## Getting started

**Requires** Power BI Desktop with *Power BI Project (.pbip) save option* and
*Store semantic model using TMDL* enabled under **Options → Preview features**.

1. Clone the repo somewhere stable.
2. Open `Recruiting_Funnel.SemanticModel/definition/expressions.tmdl` and point
   `DataFolderPath` at the `DataFiles` folder on your machine:

   ```
   expression DataFolderPath = "C:\path\to\Recruiting_Funnel_PBIP\DataFiles\" meta [...]
   ```

   Single backslashes, keep the trailing one. You can also change it after first
   open via **Transform data → Manage parameters**.
3. Open `Recruiting_Funnel.pbip`.
4. Refresh.
5. If Desktop does not do it automatically, mark `DimDate` as a date table on
   `DimDate[FullDate]` (**Table tools → Mark as date table**). Without it the
   time-intelligence measures are unreliable.

### Date parsing note

Date columns are loaded as text and parsed explicitly:

```m
#"Parsed Dates" = Table.TransformColumns(#"Changed Type",
    {{"PostedDate", each if _ = null or _ = "" then null
                         else Date.FromText ( _, [Format="yyyy-MM-dd", Culture="en-US"] ),
      type nullable date}})
```

`TransformColumnTypes({"PostedDate", type date})` parses using the query culture,
which is not the same on every machine. The explicit form is locale-proof. The
null guard matters because `OfferAcceptedDate`, `StartDate` and `ClosedDate` are
empty for open requisitions, and `ExitDate` is empty for in-process stage events.

---

## Sample data

`DataFiles/` holds generated CSVs covering January 2024 to June 2026. The funnel
is seeded with two deliberate weak points so the leakage analysis has something
real to find:

| Stage | Reached | Leakage | Days vs SLA |
|---|---:|---:|---:|
| Applied | 7,091 | 59.2% | 1.7 / 2 |
| Recruiter Screen | 2,893 | 45.7% | 3.4 / 3 |
| **Hiring Manager Review** | 1,572 | **54.9%** | **7.7 / 4** |
| Technical Assessment | 709 | 29.6% | 5.5 / 5 |
| Panel Interview | 499 | 37.9% | 7.4 / 7 |
| Final Interview | 310 | 43.5% | 6.0 / 5 |
| **Offer Extended** | 175 | 24.0% | **7.8 / 3** |
| Offer Accepted | 133 | — | — |

Other headline figures: approval lag 7.2 days, time to start 109.1 days, 31.3% of
requisitions inside their internal target, 26.4% of applications ending in
candidate withdrawal, 24.7% of losses tagged as company-caused, offer acceptance
76.0%, 20 open requisitions averaging 65.6 days old with a pipeline coverage
ratio of 3.7 candidates per open role.

**To use your own data:** replace the CSVs keeping the same file names and column
headers, and everything downstream keeps working. The columns each file needs are
visible in its partition's `Table.TransformColumnTypes` step.

---

## Repository layout

```
Recruiting_Funnel.pbip
Recruiting_Funnel.SemanticModel/
  definition/
    model.tmdl              relationships.tmdl    expressions.tmdl
    database.tmdl           cultures/en-US.tmdl
    tables/
      DimDate.tmdl          DimJob.tmdl           DimRecruiter.tmdl
      DimRejectReason.tmdl  DimSource.tmdl        DimStage.tmdl
      FactApplication.tmdl  FactRequisition.tmdl  FactStageEvent.tmdl
      _Measures.tmdl
Recruiting_Funnel.Report/
  definition/
    report.json             version.json
    pages/<page-id>/page.json
    pages/<page-id>/visuals/<visual-id>/visual.json
  StaticResources/RegisteredResources/FunnelDark.json
DataFiles/*.csv
SETUP_GUIDE.md
```

Everything is plain text, so diffs are readable and the model reviews like code.

---

## Known limitations

- `Req Age Days` uses `TODAY()` in a calculated column, so it shifts on every
  refresh. On a large model, replace it with a measure against a snapshot date.
- The report has no RLS. Add roles before sharing recruiter-level data.
- `DimDate` covers 2024-01-01 to 2026-06-30 only. Extend the CSV if your data
  falls outside that window.
- Sample data is synthetic. The patterns are plausible but the numbers are not
  from a real company.

---

## License

MIT. Use it, fork it, rip the measures out for your own model.
