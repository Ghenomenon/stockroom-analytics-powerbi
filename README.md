# Stockroom Analytics — Power BI Warehouse Diagnostic

A 5-page Power BI report diagnosing three systemic failures in a synthetic warehouse
operation of 3,204 SKUs: stockouts driven by mis-set reorder points, capital tied up
in overstock, and a warehouse layout with no demand-based slotting logic.

**[Download the exported PDF](Stockroom%20Analytics.pdf)** if you don't have Power BI
Desktop installed — every page renders statically there. The live `.pbix` supports the
slicers and cross-filtering the PDF can't.

## The business problem

- **Stockouts:** 90.8% of SKUs recorded at least one stockout last month, and 97.0% of
  SKUs sit below the reorder point a 95%-service-level policy would recommend.
- **Overstock:** £881.5K/day in total holding cost, of which 31.0% (£273K/day) is tied
  up in excess stock beyond target levels — £37.6K/day (£13.7M annualised) of that is
  on slow-moving items safe to cut without stockout risk.
- **Slotting:** Average pick time is ~95.6 seconds and statistically flat across all
  four warehouse zones and across the full popularity range (item popularity vs. pick
  time correlation: +0.01 — no relationship). High-demand items are not placed any
  closer to pick stations than low-demand items.

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
   worst-undersized SKUs sorted by reorder-point deficit.
3. **Zone 2 — Overstock** — excess holding cost by category, and a savings roadmap
   (`Safe_To_Cut`) filtered to items that can be cut with no stockout risk.
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

All four sit within noise of zero — reorder points aren't being set in response to
demand variability, stockouts aren't concentrated in slow-replenishment items any more
than fast ones, and picking efficiency has no relationship to how popular an item is.
That last point is the core evidence for the slotting finding above.

## Recommended reorder point formula

Periodic-review, 95% cycle service level (z = 1.645):

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
