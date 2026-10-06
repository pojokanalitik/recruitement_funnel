# Time-to-Fill & Recruiting Funnel Leakage — PBIP

A Power BI Project (PBIP) for recruiting analytics that goes past "how long did it
take" and answers **where in the funnel candidates are lost, how fast each stage
moves, and who owns the loss**.

## Before you open it

1. Copy the whole `Recruiting_Funnel_PBIP` folder somewhere stable.
2. Open `definition/expressions.tmdl` inside `Recruiting_Funnel.SemanticModel`
   and point `DataFolderPath` at the `DataFiles` folder on your machine:

   ```
   expression DataFolderPath = "C:\PowerBI\Recruiting_Funnel_PBIP\DataFiles\" meta [...]
   ```

   Use single backslashes and keep the trailing one. You can also change it in
   Power BI Desktop under **Transform data > Manage parameters** after first open.
3. Open `Recruiting_Funnel.pbip` in Power BI Desktop (August 2024 or later, with
   *Power BI Project (.pbip) save option* and *Store semantic model using TMDL*
   enabled under Options > Preview features).
4. Refresh.
5. Mark `DimDate` as a date table on `DimDate[FullDate]` if Desktop does not do it
   automatically (Table tools > Mark as date table).

## Model

Star schema, one grain per fact:

| Table | Grain | Rows |
|---|---|---|
| `FactRequisition` | one open role | 160 |
| `FactApplication` | one candidate application | 6,867 |
| `FactStageEvent` | one candidate entering one stage | 13,130 |
| `DimDate` | day | 912 |
| `DimStage` | funnel stage, with `StageOrder` and `SLADays` | 8 |
| `DimSource` / `DimJob` / `DimRecruiter` / `DimRejectReason` | lookups | 8 / 20 / 7 / 11 |

`FactStageEvent` is what makes leakage measurable. Without a row per stage entry
you can only count applications and hires — you cannot say *where* the funnel drained.

### Relationship notes

`DimDate` filters on **requisition posting date** by default
(`FactRequisition[PostedDateKey]`). Two other date relationships exist but are
**inactive** on purpose, to keep the model unambiguous:

- `FactApplication[AppliedDateKey]` → activate with `USERELATIONSHIP` (see
  `[Applications by Applied Date]`)
- `FactStageEvent[EnterDateKey]` → available for stage-entry trending

`FactApplication[FurthestStageID]` → `DimStage` is also inactive so the stage
slicer drives `FactStageEvent`, not the application's final stage.

## Measures

50 measures in a dedicated `_Measures` table, grouped in display folders:

| Folder | What it covers |
|---|---|
| `01 Volume` | requisition, application and hire counts |
| `02 Time to Fill` | approval lag, TTF average / median / P90, time to start, SLA |
| `03 Funnel` | reach, advance, pass-through, leakage, conversion, offer acceptance |
| `04 Stage Velocity` | days in stage vs SLA, share of total cycle time |
| `05 Dropout & Quality` | withdrawals, company-caused losses, source yield |
| `06 Pipeline Health` | open req aging, active candidates, coverage ratio |
| `07 Conditional Formatting` | colour and label measures for cards and tables |

Calculated columns on `FactRequisition`: `Days to Fill`, `Req Age Days`,
`Aging Bucket` (+ its sort column), `SLA Status`.

## Report pages

| Page | Question it answers |
|---|---|
| Executive Summary | How long, how leaky, and where do I click next |
| Time to Fill | Which of the three clocks is actually slow — approval, search, or offer-to-start |
| Funnel Leakage | Which stage destroys the most candidates, and who caused it |
| Source & Channel | Which channels deliver hires rather than applications |
| Pipeline Health | Which open roles are aging out, and is there pipeline behind them |

Page size 1920 x 1080, `FitToPage`. The dark theme lives in
`StaticResources/RegisteredResources/FunnelDark.json` — edit it once to reskin all
five pages.

## Sample data

`DataFiles/` holds generated CSVs covering Jan 2024 to Jun 2026. The funnel is
seeded with two deliberate weak points so the leakage analysis has something to
find: **Hiring Manager Review** (~54% leakage, slowest stage) and
**Offer Extended** (~22% leakage). Replace the CSVs with your ATS extract using
the same column names and everything keeps working.
