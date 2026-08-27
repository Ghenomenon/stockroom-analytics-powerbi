# Stockroom Analytics — Power BI Build Guide

This folder gives you a clean star schema exported from `logistics_dataset.csv` and
everything you need to build the report yourself in Power BI Desktop: the data
model, every DAX calculated column and measure, and a page-by-page layout plan.
No pre-solved analysis columns are included in the source files — the modeling
(relationships, calculated columns, measures) is the part you build, which is
also the part worth being able to explain in an interview.

## Files in this folder

| File | Grain | Rows |
|---|---|---|
| `Fact_Inventory.csv` | one row per SKU | 3,204 |
| `Dim_Category.csv` | one row per category | 5 |
| `Dim_Zone.csv` | one row per warehouse zone | 4 |
| `Dim_Location.csv` | one row per storage location | 100 |
| `Dim_Date.csv` | one row per calendar day (covers the `last_restock_date` range) | 365 |

---

## 1. Import & data model

1. Power BI Desktop → **Get Data → Text/CSV** → import all five files.
2. In Power Query, set data types explicitly (Power BI often guesses wrong on the
   first pass):
   - `Fact_Inventory`: `item_id` (Text), `CategoryKey`/`LocationKey` (Text),
     `last_restock_date` (Date), everything else numeric (Decimal Number, except
     `stockout_count_last_month` / `total_orders_last_month` → Whole Number).
   - `Dim_Location`: `LocationKey` (Text), `LocationNumber`/`ZoneKey` (Whole Number).
   - `Dim_Date`: `DateKey` (Date), `Year`/`MonthNumber`/`Day` (Whole Number).
3. Close & Apply, then go to **Model view** and build these relationships
   (all single-direction, one-to-many, dimension → fact):

```
Dim_Category[CategoryKey]  1 ──< * Fact_Inventory[CategoryKey]
Dim_Location[LocationKey]  1 ──< * Fact_Inventory[LocationKey]
Dim_Location[ZoneKey]      * >── 1 Dim_Zone[ZoneKey]
Dim_Date[DateKey]          1 ──< * Fact_Inventory[last_restock_date]
```

That's a proper star schema: `Fact_Inventory` in the middle, three dimensions
plus a snowflaked `Dim_Zone` hanging off `Dim_Location` (zone is a property of
a location, not of the item directly — modeling it this way is what lets you
slice by zone *or* by individual location bay).

---

## 2. Calculated columns to add

Add these on `Fact_Inventory` (Modeling → New Column). They turn the raw fields
into the fields the report actually needs.

**Demand during lead time**
```DAX
DLT_Mean = Fact_Inventory[daily_demand] * Fact_Inventory[lead_time_days]
```
```DAX
DLT_StdDev = Fact_Inventory[demand_std_dev] * SQRT(Fact_Inventory[lead_time_days])
```

**Recommended reorder point** (periodic-review, 95% cycle service level —
z = 1.645; swap to 2.05 for a 99% target on A-items)
```DAX
Exposure_Days = Fact_Inventory[lead_time_days] + Fact_Inventory[reorder_frequency_days]
```
```DAX
Recommended_ROP =
Fact_Inventory[daily_demand] * Fact_Inventory[Exposure_Days]
    + 1.645 * Fact_Inventory[demand_std_dev] * SQRT(Fact_Inventory[Exposure_Days])
```
```DAX
ROP_Deficit = Fact_Inventory[Recommended_ROP] - Fact_Inventory[reorder_point]
```
```DAX
Service_Level_Band =
VAR z = DIVIDE(Fact_Inventory[reorder_point] - Fact_Inventory[DLT_Mean], Fact_Inventory[DLT_StdDev])
RETURN
    SWITCH(
        TRUE(),
        z < 0, "1. Below lead-time demand",
        z < 0.84, "2. <60% service level",
        z < 1.28, "3. 60-90%",
        z < 1.645, "4. 90-95%",
        "5. >=95% service level"
    )
```

**Overstock**
```DAX
Target_Stock = Fact_Inventory[reorder_point] + Fact_Inventory[daily_demand] * Fact_Inventory[reorder_frequency_days]
```
```DAX
Excess_Units = MAX(0, Fact_Inventory[stock_level] - Fact_Inventory[Target_Stock])
```
```DAX
Excess_Daily_Cost = Fact_Inventory[Excess_Units] * Fact_Inventory[holding_cost_per_unit_day]
```
```DAX
Weeks_Of_Cover = DIVIDE(Fact_Inventory[stock_level], DIVIDE(Fact_Inventory[forecasted_demand_next_7d], 7)) / 7
```
```DAX
Safe_To_Cut =
IF(
    Fact_Inventory[Excess_Units] > 0
        && Fact_Inventory[stockout_count_last_month] <= 2
        && Fact_Inventory[turnover_ratio] < 8,
    "Safe to cut",
    "Hold / needs review"
)
```

**Slotting**
```DAX
Popularity_Rank = RANKX(ALL(Fact_Inventory), Fact_Inventory[item_popularity_score],, DESC)
```
```DAX
Velocity_Tier =
IF(
    Fact_Inventory[Popularity_Rank] <= COUNTROWS(ALL(Fact_Inventory)) * 0.2,
    "A - Top 20%",
    IF(Fact_Inventory[Popularity_Rank] <= COUNTROWS(ALL(Fact_Inventory)) * 0.5, "B - Mid 30%", "C - Bottom 50%")
)
```

---

## 3. Measures to add

Group these into a display folder per page (right-click each measure → *Display
folder* → `Zone 1 Stockouts` / `Zone 2 Overstock` / `Zone 3 Slotting` /
`Overview`) — an interviewer looking at your fields pane should immediately see
the report is organised, not a flat list.

### Overview
```DAX
Total SKUs = COUNTROWS(Fact_Inventory)
```
```DAX
% SKUs With Stockout =
DIVIDE(CALCULATE(COUNTROWS(Fact_Inventory), Fact_Inventory[stockout_count_last_month] > 0), [Total SKUs])
```
```DAX
Total Daily Holding Cost = SUMX(Fact_Inventory, Fact_Inventory[stock_level] * Fact_Inventory[holding_cost_per_unit_day])
```

### Zone 1 — Stockouts
```DAX
Avg Stockouts = AVERAGE(Fact_Inventory[stockout_count_last_month])
```
```DAX
Avg Fulfillment Rate = AVERAGE(Fact_Inventory[order_fulfillment_rate])
```
```DAX
Avg Reorder Point = AVERAGE(Fact_Inventory[reorder_point])
```
```DAX
Avg DLT Mean = AVERAGE(Fact_Inventory[DLT_Mean])
```
```DAX
% SKUs Below Recommended ROP =
DIVIDE(CALCULATE(COUNTROWS(Fact_Inventory), Fact_Inventory[ROP_Deficit] > 0), [Total SKUs])
```

**Generic Pearson-correlation pattern** — reuse this shape for every
correlation on the report (swap the two column references):
```DAX
Corr ROP vs Daily Demand =
VAR AvgX = AVERAGE(Fact_Inventory[reorder_point])
VAR AvgY = AVERAGE(Fact_Inventory[daily_demand])
VAR SumXY = SUMX(Fact_Inventory, (Fact_Inventory[reorder_point] - AvgX) * (Fact_Inventory[daily_demand] - AvgY))
VAR SumX2 = SUMX(Fact_Inventory, (Fact_Inventory[reorder_point] - AvgX) ^ 2)
VAR SumY2 = SUMX(Fact_Inventory, (Fact_Inventory[daily_demand] - AvgY) ^ 2)
RETURN DIVIDE(SumXY, SQRT(SumX2 * SumY2))
```
Build three more copies of this measure for: `Corr Stockouts vs ROP`,
`Corr Stockouts vs Lead Time`, and (for Zone 3) `Corr Popularity vs Pick Time`.

### Zone 2 — Overstock
```DAX
Total Excess Units = SUM(Fact_Inventory[Excess_Units])
```
```DAX
Total Excess Daily Cost = SUM(Fact_Inventory[Excess_Daily_Cost])
```
```DAX
% Holding Cost In Excess = DIVIDE([Total Excess Daily Cost], [Total Daily Holding Cost])
```
```DAX
Safe To Cut Daily Savings =
CALCULATE([Total Excess Daily Cost], Fact_Inventory[Safe_To_Cut] = "Safe to cut")
```
```DAX
Safe To Cut Annualised = [Safe To Cut Daily Savings] * 365
```
```DAX
% SKUs Low Turnover =
DIVIDE(CALCULATE(COUNTROWS(Fact_Inventory), Fact_Inventory[turnover_ratio] < 5), [Total SKUs])
```

### Zone 3 — Slotting
```DAX
Avg Pick Time = AVERAGE(Fact_Inventory[picking_time_seconds])
```
```DAX
Avg Layout Efficiency = AVERAGE(Fact_Inventory[layout_efficiency_score])
```
```DAX
% Top20 In Zone =
VAR ZoneTop20 = CALCULATE(COUNTROWS(Fact_Inventory), Fact_Inventory[Velocity_Tier] = "A - Top 20%")
VAR AllTop20 = CALCULATE(COUNTROWS(Fact_Inventory), Fact_Inventory[Velocity_Tier] = "A - Top 20%", ALL(Dim_Zone))
RETURN DIVIDE(ZoneTop20, AllTop20)
```

---

## 4. Report pages

**Page 1 — Overview.** KPI cards: Total SKUs, % SKUs With Stockout, Total Daily
Holding Cost, Avg Pick Time. Slicers across the top: Category, Zone. A short
text box stating the three findings in one line each (this is your "so what" —
write it yourself once the numbers render).

**Page 2 — Zone 1: Stockouts.** Clustered column chart of `Service_Level_Band`
(axis) vs `Total SKUs`/count (this is the "implied service level" distribution).
A card row for the four correlation measures. A table: `item_id`, `category`,
`daily_demand`, `lead_time_days`, `reorder_point`, `Recommended_ROP`,
`stockout_count_last_month` — sort by `ROP_Deficit` descending to surface the
worst-undersized SKUs first.

**Page 3 — Zone 2: Overstock.** Bar chart: `CategoryName` vs
`Total Excess Daily Cost`. KPI cards: `% Holding Cost In Excess`,
`Safe To Cut Daily Savings`, `Safe To Cut Annualised`. Table filtered to
`Safe_To_Cut = "Safe to cut"`, sorted by `Excess_Daily_Cost` descending.

**Page 4 — Zone 3: Slotting.** Bar chart: `ZoneName` vs `Avg Pick Time`
(zero-based axis — flat bars are the finding). Second bar chart:
`ZoneName` vs `% Top20 In Zone`, with a manually-added reference line at 25%
(Format pane → *Add a constant line*) to show what "no slotting logic" looks
like against what random placement would produce. A scatter chart:
`item_popularity_score` (X) vs `picking_time_seconds` (Y) — this is the
clearest single visual for "these are unrelated," and it's worth including
precisely because the cloud looks random.

**Page 5 — Data Model (optional, strong for interviews).** A screenshot or a
`Model view` export showing the star schema. Recruiters who work with data
tend to check whether a candidate modeled the data properly before dashboarding
it — this page answers that before they have to ask.

---

## 5. Talking about it

Be ready to explain, unprompted: why a star schema over a single flat table;
why the reorder-point formula uses `lead_time + review_period` rather than
just lead time (periodic vs continuous review); and why correlation is
computed manually in DAX rather than via a built-in function (DAX has no
native `CORREL()` — this is a legitimate, slightly advanced pattern worth
having memorised, not just pasted in).
