# Dashboard redesign — matching the reference screenshot

This is a companion to `BUILD_GUIDE.md` / `STEP_BY_STEP.md`. It doesn't replace them —
it layers a visual redesign (sidebar nav, icon KPI cards, donut, heatmap matrix) on top
of the model and measures you're already building, and adds the handful of new columns
and measures needed to hit every number on the reference image.

Follow it after you've completed Phase 5 (all measures built) in `STEP_BY_STEP.md`.

---

## 1. What changes and why (read this first)

The reference image is one page — **Overview** — from a report whose sidebar shows
four sections: Overview, Replenishment & Demand, Location Performance, Item Detail.
Your project's guide currently plans five pages: Overview, Zone 1 Stockouts, Zone 2
Overstock, Zone 3 Slotting, Data Model. Here's how they reconcile, and why:

| Reference nav | Maps to your existing plan | Adjustment |
|---|---|---|
| **Overview** | Overview | Full redesign per §4 below — icon cards, donut, heatmap |
| **Replenishment & Demand** | Zone 1 — Stockouts (renamed) | Recruiter-friendlier name for the same content |
| **Location Performance** | Zone 3 — Slotting (renamed) | Same page, broadened with the location risk table |
| **Item Detail** | *(new — didn't exist)* | Drillthrough page, one row → one SKU's full profile |
| *(not in reference)* | Zone 2 — Overstock | **Kept as a 5th page.** See below. |
| *(not in reference)* | Data Model | **Kept as a 6th, optional page.** Unchanged. |

**Why keep Overstock even though the reference doesn't show it:** it's real analytical
work your guide already specs (`Target_Stock`, `Excess_Units`, `Safe_To_Cut`) and it's
one of your three interview talking points. The reference dashboard is simply a
narrower slice — matching its exact page count would mean deleting a page's worth of
your own analysis to imitate a screenshot. Add it as a 5th sidebar button instead.

**The one number-level problem in the reference, and the fix:** the donut chart's five
segments (818 + 240 + 620 + 870 + 1,486 = 4,034 items) add up to **126% of the 3,204
total** — the segments overlap (an item can be both "Lead-Time Risk" and "7D Forecast
Risk" at once). A donut/pie encodes parts of a whole; showing overlapping categories in
one is mathematically incoherent, not just a style nitpick — the slice angles imply
percentages that don't sum to 100%, so the chart is lying by construction. §5 below
redefines the same five ideas as a **priority-ordered, mutually exclusive** segment so
the donut is honest and still visually close to the original. If you'd rather preserve
the literal overlapping-flag view, §5 also gives you that as a row of independent stat
tiles instead — pick one, don't do both on the same page.

---

## 2. New calculated columns (add to `Fact_Inventory`)

Add these *after* the 13 columns already in `BUILD_GUIDE.md` §2 (they build on
`DLT_Mean` and `ROP_Deficit`, so those must already exist).

**Lead-time stock gap** — the reference's literal "Lead-Time Demand" / "Lead-Time
Stock Gap" columns, using raw lead-time demand with no safety stock. This is
deliberately simpler than `Recommended_ROP`/`ROP_Deficit` (which include a safety
margin and the review period) — keep both: this one is the plain-English "will this
guaranteed run out before the next delivery arrives" framing for the Overview page;
`ROP_Deficit` stays the more rigorous version on the Replenishment & Demand page.

```DAX
Lead_Time_Stock_Gap = Fact_Inventory[reorder_point] - Fact_Inventory[DLT_Mean]
```

**Binary label for the 100%-stacked category chart** (§4, chart 2):
```DAX
Lead_Time_Risk_Label = IF(Fact_Inventory[Lead_Time_Stock_Gap] < 0, "At Risk", "Covered")
```

**Mutually exclusive risk segment** for the donut (§5). Order matters — each item
falls into the *first* bucket it matches, most structurally urgent first:
```DAX
Risk_Segment =
SWITCH(
    TRUE(),
    Fact_Inventory[Lead_Time_Stock_Gap] < 0, "1. Lead-Time Risk",
    Fact_Inventory[stock_level] <= Fact_Inventory[reorder_point], "2. Reorder Due",
    Fact_Inventory[forecasted_demand_next_7d] > Fact_Inventory[stock_level], "3. 7D Forecast Risk",
    Fact_Inventory[ROP_Deficit] > 0, "4. Reorder Blind Spot",
    "5. Covered (No Risk)"
)
```
Read this as a priority ladder: *(1)* the reorder point itself is set below lead-time
demand — broken by policy design, regardless of current stock. *(2)* stock has already
fallen to/below the reorder point — action needed today. *(3)* forecasted 7-day demand
alone will exceed what's on the shelf. *(4)* none of the harder tripwires fired, but
the more rigorous `Recommended_ROP` (which includes safety stock) still says this item
is under-covered — the "blind spot" the simple reorder rule misses. *(5)* everything
else.

---

## 3. New measures (add to `_Measures`, set Display folder per column below)

| Measure | DAX | Folder | Format |
|---|---|---|---|
| `Total Inventory Value` | `SUMX(Fact_Inventory, Fact_Inventory[stock_level] * Fact_Inventory[unit_price])` | Overview | `$#,##0,,"M"` |
| `Total Stock Units` | `SUM(Fact_Inventory[stock_level])` | Overview | `#,##0` |
| `Total Daily Demand` | `SUM(Fact_Inventory[daily_demand])` | Overview | `#,##0` |
| `Total 7D Forecast Demand` | `SUM(Fact_Inventory[forecasted_demand_next_7d])` | Overview | `#,##0` |
| `Stock to 7D Forecast Ratio` | `DIVIDE([Total Stock Units], [Total 7D Forecast Demand])` | Overview | `0.0%` |
| `Days Of Demand Coverage` | `DIVIDE([Total Stock Units], [Total Daily Demand])` | Overview | `#,##0` |
| `Total Stockout Events` | `SUM(Fact_Inventory[stockout_count_last_month])` | Overview | `#,##0` |
| `% Lead-Time Risk` | `DIVIDE(CALCULATE(COUNTROWS(Fact_Inventory), Fact_Inventory[Lead_Time_Stock_Gap] < 0), [Total SKUs])` | Overview | `0.0%` |
| `Reorder Items Count` | `CALCULATE(COUNTROWS(Fact_Inventory), Fact_Inventory[stock_level] <= Fact_Inventory[reorder_point])` | Overview | `#,##0` |
| `% Reorder Items` | `DIVIDE([Reorder Items Count], [Total SKUs])` | Overview | `0.0%` |
| `Lead-Time Shortfall Units` | `SUMX(FILTER(Fact_Inventory, Fact_Inventory[Lead_Time_Stock_Gap] < 0), -Fact_Inventory[Lead_Time_Stock_Gap])` | Overview | `#,##0` |
| `7D Forecast Shortfall Units` | `SUMX(FILTER(Fact_Inventory, Fact_Inventory[forecasted_demand_next_7d] > Fact_Inventory[stock_level]), Fact_Inventory[forecasted_demand_next_7d] - Fact_Inventory[stock_level])` | Overview | `#,##0` |
| `Reorder Blind Spot Callout` | `"⚠ " & FORMAT(CALCULATE(COUNTROWS(Fact_Inventory), Fact_Inventory[Risk_Segment] = "4. Reorder Blind Spot"), "#,##0") & " items are above their reorder point but forecasted to stock out within 7 days."` | Overview | Text |

`Total SKUs` and `% SKUs With Stockout` already exist from `BUILD_GUIDE.md` §3 — reuse
them directly for the "Stock Units"-adjacent "Items with At Least One Stockout" card
and the KPI row; you don't need duplicates.

`% Lead-Time Risk` is written with `CALCULATE`/`COUNTROWS` rather than a plain column
filter so it stays correct at every grain you'll use it at (whole report, one category,
one location, or a location×category cell in the matrix) — it responds to whatever
filter context the visual puts it in, which a hardcoded row count would not.

---

## 4. Overview page — visual-by-visual build

### Sidebar navigation
Use the built-in **Page navigator** visual (Visualizations pane, not a manual button
row) — it auto-generates one entry per report page and stays in sync when you rename,
reorder, or add pages, which a hand-built set of bookmark buttons won't do for free.
1. Insert a tall rectangle (Insert → Shapes) down the left edge, fill it navy
   (`#0B1E3D` — pick anything consistent with your theme from Phase 7).
2. Insert → **Page navigator**, resize to fill the rectangle, orientation Vertical
   (Format pane → Layout).
3. Format pane → Icons: on, and assign one icon per page (gauge/speedometer for
   Overview, cart/refresh for Replenishment & Demand, warehouse for Location
   Performance, box for Item Detail) — pick from Power BI's built-in icon set via
   Insert → Icons if the navigator's auto-icon doesn't have a good match, then swap
   the navigator's icon reference to the one you inserted.
4. Hide the Overstock and Data Model pages from the navigator if you want the sidebar
   to show only the four reference tabs (Page navigator → Format → "Pages" lets you
   exclude specific pages) — they're still reachable from the page tab strip.

### Filter bar
Three slicers, all set to **Dropdown** (Format pane → Slicer settings → Style):
`Dim_Category[CategoryName]`, `Dim_Location[LocationKey]`, `Risk_Segment`. Turn on
**View → Sync slicers** and sync all three across every page — this is a deliberate
addition beyond the reference (its screenshot can't show cross-page behavior), and it's
what makes the drillthrough-free pages actually behave like one dashboard instead of
four disconnected ones.

### KPI card row (8 cards)
Use the **Card** visual (New) — its Style options include an icon slot, so you don't
need to hand-build icon+textbox groups.

| Card | Measure | Icon |
|---|---|---|
| Inventory Value | `[Total Inventory Value]` | stacked coins |
| Stock Units | `[Total Stock Units]` | box |
| Fulfillment Rate | `[Avg Fulfillment Rate]` (existing) | check-circle |
| Lead-Time Risk | `[% Lead-Time Risk]` | warning-triangle |
| Reorder Items | `[Reorder Items Count]`, subtitle `[% Reorder Items]` | cart |
| Stockouts | `[Total Stockout Events]` | alert-circle |
| Stock Coverage | `[Days Of Demand Coverage]` | calendar |

Color the icon/value red for Lead-Time Risk and Stockouts, amber for Reorder Items,
green for Fulfillment Rate, neutral for the rest — status color carries meaning here,
so don't reuse your categorical theme's colors for these.

### Chart row (3 charts)
1. **Stock vs 7-Day Forecast Demand by Category** — Clustered column. Axis
   `Dim_Category[CategoryName]`, values `[Total Stock Units]` and
   `[Total 7D Forecast Demand]`.
2. **Lead-Time Risk by Category** — 100% Stacked column. Axis
   `Dim_Category[CategoryName]`, legend `Lead_Time_Risk_Label`, value = count of
   `item_id`. (Using the label column as legend and letting the visual normalize to
   100% is simpler and less error-prone than computing two percentage measures by
   hand.)
3. **Top 10 Locations by Lead-Time Risk %** — Bar chart (horizontal). Axis
   `Dim_Location[LocationKey]`, value `[% Lead-Time Risk]`, then Filters pane → Top N
   → 10, by `[% Lead-Time Risk]`. Sort descending.

### Bottom-left: Risk Segment donut
Donut chart. Legend `Risk_Segment`, values = count of `item_id` (or `[Total SKUs]`,
same result once filtered to the visual's legend groups). Because `Risk_Segment` is
mutually exclusive (§2), the slice percentages now sum to exactly 100% — the honest
version of the reference's chart. Add a **Card** visual beneath it bound to
`[Reorder Blind Spot Callout]` for the explanatory line the reference shows as a
tooltip box.

*(If you specifically want the reference's overlapping-flags view instead of the
fixed donut, replace this visual with four independent stat tiles — one each for
"% Lead-Time Risk", "% Reorder Items", a new `% Reorder Blind Spot` measure, and a new
`% 7D Forecast Risk` measure, each computed independently with its own `CALCULATE`.
Don't put overlapping percentages in a donut.)*

### Bottom-middle: Risk by Location & Category
**Matrix** visual. Rows `Dim_Location[LocationKey]`, columns
`Dim_Category[CategoryName]`, values `[% Lead-Time Risk]`. Format pane → Conditional
formatting → Background color → apply to the value field, 3-color scale (green low →
amber mid → red high, thresholds 0% / 25% / 50%+). This is a sequential/diverging
magnitude encoding, not a categorical one — one color ramp, light-to-dark, exactly as
`dataviz` calls for.

### Bottom-right: Portfolio Overview mini-cards
Six small Cards: `[Total Daily Demand]`, `[Total 7D Forecast Demand]`,
`[Stock to 7D Forecast Ratio]`, `[Lead-Time Shortfall Units]`,
`[7D Forecast Shortfall Units]`, `[% SKUs With Stockout]` (relabel its card title
"Items with ≥1 Stockout").

### Bottom table: Top At-Risk Items
Table visual: `item_id`, `Dim_Category[CategoryName]`, `Dim_Location[LocationKey]`,
`stock_level`, `reorder_point`, `daily_demand`, `lead_time_days`, `DLT_Mean` (rename
display to "Lead-Time Demand"), `Lead_Time_Stock_Gap` (rename display to "Lead-Time
Stock Gap"), `forecasted_demand_next_7d`, `stockout_count_last_month`,
`order_fulfillment_rate`, `KPI_score`. Sort by `Lead_Time_Stock_Gap` ascending (most
negative — worst gap — first). Conditional formatting → Background color on
`Lead_Time_Stock_Gap`: red for negative values, matching the reference's highlighted
cells.

---

## 5. Other pages (brief — reuse your existing plans)

- **Replenishment & Demand** (renamed Zone 1): exactly as `BUILD_GUIDE.md` §4 Page 2 —
  `Service_Level_Band` chart, the four correlation-measure cards, the ROP_Deficit
  table. Add the `Lead_Time_Stock_Gap` column to that table too so the framing is
  consistent with Overview.
- **Location Performance** (renamed Zone 3): as `BUILD_GUIDE.md` §4 Page 4, plus add
  the same Location × Category risk matrix from Overview (§4) filtered/pinned to
  whichever location a user has clicked into — this is what makes it feel like a
  drill-down rather than a duplicate page.
- **Item Detail** (new): set up as a **drillthrough page** — Format pane → Drillthrough
  → add field `item_id`. Right-click any row in the Overview table → Drill through →
  Item Detail. Layout: a card row repeating that one SKU's key fields (stock, ROP,
  lead time, forecast, KPI_score), plus its `Risk_Segment` badge and its position on
  the `item_popularity_score` vs `picking_time_seconds` scatter (highlight the point).
- **Overstock & Excess** (kept, Zone 2): unchanged from `BUILD_GUIDE.md` §4 Page 3.
- **Data Model**: unchanged, optional bonus page.

---

## 6. QA additions (extend `STEP_BY_STEP.md` Phase 8's table)

| Measure | Expected (approx., will vary with your real data) |
|---|---|
| `[Total Inventory Value]` | in the tens of millions — sanity-check against `SUM(stock_level)` × average `unit_price` |
| `[% Lead-Time Risk]` | should be noticeably lower than `[% SKUs Below Recommended ROP]` (~97%), since it has no safety-stock margin — if it comes out *higher*, a sign got flipped in `Lead_Time_Stock_Gap` |
| `Risk_Segment` slice percentages | must sum to exactly 100% — if they don't, the `SWITCH` isn't exhaustive (check the final `"5. Covered"` fallback is reachable) |
