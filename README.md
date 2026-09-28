# Stockroom Analytics — Power BI Warehouse Diagnostic

A 5-page Power BI diagnostic exploring stock availability, inventory-cost scenarios
and picking performance across a synthetic snapshot of 3,204 SKUs. The findings
identify questions to investigate; they do not establish root causes or demonstrate
implemented operational improvements or financial savings.

**[Download the exported PDF](Stockroom%20Analytics.pdf)** if you don't have Power BI
Desktop installed — every page renders statically there. The live `.pbix` supports the
slicers and cross-filtering the PDF can't.

## Findings to investigate and model scenarios

- **Stock availability:** 90.8% of SKUs have a recorded stockout in the supplied
  snapshot. About 97.0% fall below an illustrative replenishment target using a
  95% cycle-service-level assumption. This flags policy assumptions for review;
  it does not establish that the current thresholds caused stockouts.
- **Inventory-cost scenario:** Synthetic daily holding-cost inputs produce a total
  of £881.5K/day. Under the report's assumptions, 31.0% (£273K/day) is associated
  with stock above model targets. A slow-moving subset accounts for £37.6K/day
  (£13.7M when annualised). These are model calculations, not validated savings
  or realistic business cost estimates. The historical `Safe_To_Cut` label marks
  review candidates; it does not establish that reductions are safe or risk-free.
  Validate cost units, demand variation, lead times and service levels before
  recommending inventory changes.
- **Picking performance:** Average pick time is approximately 95.6 seconds.
  Popularity and pick time have a near-zero linear correlation (+0.01).
  This does not establish a layout problem or show how far items are from pick
  stations. Investigate pick-path distance, order-line frequency and congestion
  before proposing slotting changes.

The original report and export retain their historical labels. Read them alongside
these limitations and the [portfolio case study](https://chigozie-nkwopara.netlify.app/warehouse-case-study).

## Data model

Star schema, `Fact_Inventory` (one row per SKU) at the center, three dimension tables
plus a snowflaked zone table:

```
Dim_Category[CategoryKey]  1 ──< * Fact_Inventory[CategoryKey]
Dim_Location[LocationKey]  1 ──< * Fact_Inventory[LocationKey]
Dim_Location[ZoneKey]      * >── 1 Dim_Zone[ZoneKey]
Dim_Date[DateKey]          1 ──< * Fact_Inventory[last_restock_date]
```

Zone is modeled as a property of a storage location rather than of the item directly,
which is what lets the report slice by zone or by individual location bay.

| File | Grain | Rows |
|---|---|---|
| `Fact_Inventory.csv` | one row per SKU | 3,204 |
| `Dim_Category.csv` | one row per category | 5 |
| `Dim_Zone.csv` | one row per warehouse zone | 4 |
| `Dim_Location.csv` | one row per storage location | 100 |
| `Dim_Date.csv` | one row per calendar day | 365 |

## Report pages

1. **Overview** — portfolio KPIs (Total SKUs, % SKUs With Stockout, Total Daily
   Holding Cost, Avg Pick Time) with Category/Zone slicers.
2. **Zone 1 — Stockouts** — implied service-level distribution
   (`Service_Level_Band`), four Pearson-correlation cards, and a table of the
   SKUs with the largest gap below the illustrated replenishment target.
3. **Zone 2 — Overstock** — modelled excess holding cost by category and a
   scenario filter (`Safe_To_Cut`) identifying candidates for further review,
   subject to demand, service-level and stockout-risk validation.
4. **Zone 3 — Slotting** — pick time and top-20%-velocity placement by zone (both
   flat, which is the finding), plus a popularity-vs-pick-time scatter with trend
   line.
5. **Data Model** — the star schema and table-grain documentation.

## Notable technique: correlation in DAX

DAX has no built-in `CORREL()`. All four correlation measures use the manual Pearson
formula:

```DAX
Corr X vs Y =
VAR AvgX = AVERAGE(Fact_Inventory[x])
VAR AvgY = AVERAGE(Fact_Inventory[y])
VAR SumXY = SUMX(Fact_Inventory, (Fact_Inventory[x] - AvgX) * (Fact_Inventory[y] - AvgY))
VAR SumX2 = SUMX(Fact_Inventory, (Fact_Inventory[x] - AvgX) ^ 2)
VAR SumY2 = SUMX(Fact_Inventory, (Fact_Inventory[y] - AvgY) ^ 2)
RETURN DIVIDE(SumXY, SQRT(SumX2 * SumY2))
```

All four measures were cross-checked against the raw CSV independently of Power BI to
confirm the DAX evaluates correctly:

| Measure | Value |
|---|---|
| `Corr ROP vs Daily Demand` | −0.01 |
| `Corr Stockouts vs Lead Time` | −0.02 |
| `Corr Stockouts vs ROP` | +0.03 |
| `Corr Popularity vs Pick Time` | +0.01 |

These coefficients show little linear association between the selected variables
in this snapshot. They do not establish how policies were set, what caused
stockouts, or whether layout changes would improve picking. Transaction history
and operational measurements are needed to test those hypotheses.

## Illustrative periodic-review target

The report's `Recommended_ROP` measure represents an illustrative order-up-to
target for periodic review, rather than a universal reorder trigger. It assumes
stable lead times and independent daily demand variation, with a 95% cycle
service level (z = 1.645). An order quantity would also need inventory on order,
backorders and purchasing constraints:

```DAX
Recommended_ROP =
daily_demand * (lead_time_days + reorder_frequency_days)
    + 1.645 * demand_std_dev * SQRT(lead_time_days + reorder_frequency_days)
```

Using `lead_time + review_period` rather than lead time alone matters here because
this is a **periodic-review** policy (stock is checked every `reorder_frequency_days`,
not continuously) — between two review points, a shortfall isn't caught until the next
check, so the exposure window is the full cycle, not just the supplier's lead time.

## Tools

Power BI Desktop, DAX, Power Query. Source data is a synthetic logistics dataset — no
real company or customer data.

## What I'd do differently

This is a diagnostic of a single synthetic inventory snapshot. I would use transaction-level history to measure demand variability and test the replenishment policy over time, incorporating supplier lead-time variation into the safety-stock calculation.

Before recommending inventory reductions, I would segment SKUs by demand and value, test several service levels, and estimate both stockout risk and carrying-cost changes. The report's 'safe to cut' label is a model scenario, not a validated operational saving.

For slotting, a near-zero correlation does not establish that the warehouse layout causes slow picks. I would collect order-line frequency, pick paths, travel distance and congestion by location, then compare a proposed layout against measured travel time before rollout.
