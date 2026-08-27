# Step-by-step: building the Stockroom Analytics report

Follow this in order — later steps depend on earlier ones (relationships before
calculated columns, calculated columns before the measures that reference them).
Full DAX code for every formula named below is in `BUILD_GUIDE.md` — this file
is the click-by-click sequence; that one is the reference you copy formulas from.

Budget: roughly 3-5 hours spread over a few sessions is normal for a first build
of this size. Don't rush the modeling phase (Steps 1-14) — a wrong relationship
there causes confusing errors much later that are hard to trace back.

---

## Phase 0 — Setup (5 min)

1. Install **Power BI Desktop** — free, from the Microsoft Store or
   powerbi.microsoft.com/desktop. If it's already installed, open it and check
   **Help → About** for a recent version (monthly updates matter for newer DAX
   functions).
2. Confirm the five CSVs and `BUILD_GUIDE.md` are in
   `Downloads\stockroom-powerbi-project\`.
3. On first launch, close the startup screen and save a blank file now as
   `Stockroom Analytics.pbix` in that same folder — save again every 15-20
   minutes as you go (Ctrl+S), Power BI Desktop has no autosave.

## Phase 1 — Import the five tables (15 min)

4. **Home → Get Data → Text/CSV**.
5. Select `Fact_Inventory.csv`. In the preview window click **Transform Data**
   (not Load) — this opens Power Query Editor so you can fix types before
   anything hits the model.
6. Repeat step 4-5 for `Dim_Category.csv`, `Dim_Zone.csv`, `Dim_Location.csv`,
   `Dim_Date.csv`. All five should now be open as queries in Power Query
   Editor, listed on the left.

## Phase 2 — Fix data types in Power Query (15 min)

Do this per query — click the query name on the left, then click each column
header's type icon (the little `ABC` or `1.2` symbol) to change it.

7. **Fact_Inventory**: `item_id` → Text, `CategoryKey` → Text, `LocationKey` →
   Text, `last_restock_date` → Date, `stockout_count_last_month` → Whole
   Number, `total_orders_last_month` → Whole Number, everything else numeric →
   Decimal Number.
8. **Dim_Location**: `LocationKey` → Text, `LocationNumber` → Whole Number,
   `ZoneKey` → Whole Number.
9. **Dim_Date**: `DateKey` → Date, `Year`/`MonthNumber`/`Day` → Whole Number,
   the rest → Text.
10. **Dim_Category** / **Dim_Zone**: key columns → Whole Number, name columns
    → Text (these usually import correctly already — just verify).
11. **Home → Close & Apply**. Power BI loads all five tables into the model.

## Phase 3 — Build the star schema relationships (10 min)

12. Click the **Model view** icon on the far-left sidebar (looks like three
    connected boxes).
13. Drag the tables apart so you can see all five clearly — put
    `Fact_Inventory` in the middle, the four `Dim_` tables around it.
14. Draw each relationship by dragging the key column in one table onto the
    matching key column in the other:
    - `Dim_Category[CategoryKey]` → `Fact_Inventory[CategoryKey]`
    - `Dim_Location[LocationKey]` → `Fact_Inventory[LocationKey]`
    - `Dim_Location[ZoneKey]` → `Dim_Zone[ZoneKey]`
    - `Dim_Date[DateKey]` → `Fact_Inventory[last_restock_date]`

    For each one, a dialog confirms cardinality and direction — accept the
    default **one-to-many, single direction, dimension → fact** for all four.
    You should end up with four lines connecting the star, each showing `1` on
    the dimension side and `*` on the fact side.

## Phase 4 — Calculated columns (30-40 min)

15. Switch to **Table view** (grid icon, left sidebar), select
    `Fact_Inventory` in the fields pane.
16. **Modeling → New Column**, and add the following *in this order* (some
    reference earlier ones — adding out of order will error until the
    dependency exists). Copy each formula from `BUILD_GUIDE.md` §2:
    1. `DLT_Mean`
    2. `DLT_StdDev`
    3. `Exposure_Days`
    4. `Recommended_ROP`
    5. `ROP_Deficit`
    6. `Service_Level_Band`
    7. `Target_Stock`
    8. `Excess_Units`
    9. `Excess_Daily_Cost`
    10. `Weeks_Of_Cover`
    11. `Safe_To_Cut`
    12. `Popularity_Rank`
    13. `Velocity_Tier`

    After each one, glance at the column values in the grid — a column of
    blanks or errors means the formula or a dependency is wrong; fix it before
    moving to the next.

## Phase 5 — Set up a Measures table, then add measures (30-40 min)

17. **Home → Enter Data**, leave it empty (one dummy column), name the table
    `_Measures`, click **Load**. Keeping measures in their own empty table
    (rather than scattered across `Fact_Inventory`) is standard practice and
    signals you know it — every measure below lives here, not on the fact
    table.
18. Select `_Measures` in the fields pane. **Modeling → New Measure**, and add
    each measure from `BUILD_GUIDE.md` §3, one at a time. After creating each,
    right-click it → **Properties** (or use the Modeling ribbon) to:
    - Set the **Display folder** (`Overview`, `Zone 1 Stockouts`,
      `Zone 2 Overstock`, or `Zone 3 Slotting`) so the fields pane is
      organised by page.
    - Set the format: percentages as `%` with 1 decimal, money as `£` with 0
      decimals, correlation values as decimal with 2-3 decimals.
19. Build order: Overview measures first, then Zone 1, Zone 2, Zone 3 — same
    order as the guide. For the four correlation measures, build
    `Corr ROP vs Daily Demand` completely first, confirm it returns a sensible
    number (should land near 0), then duplicate it three times and swap the
    column references for the other three pairs.

## Phase 6 — Build the pages (60-90 min)

Work one page at a time; rename each page tab as you go (double-click the tab).

20. **Page 1 — Overview.** Insert 4 **Card** visuals: `[Total SKUs]`,
    `[% SKUs With Stockout]`, `[Total Daily Holding Cost]`, `[Avg Pick Time]`.
    Add two **Slicer** visuals along the top: `Dim_Category[CategoryName]` and
    `Dim_Zone[ZoneName]`. Add a text box underneath with a one-line summary per
    problem area — write this yourself once you see your own numbers render.

21. **Page 2 — Zone 1: Stockouts.** Clustered column chart: axis =
    `Service_Level_Band`, value = count of `item_id`. Four cards for the
    correlation measures in a row. A table visual with columns `item_id`,
    `category` (via `Dim_Category[CategoryName]`), `daily_demand`,
    `lead_time_days`, `reorder_point`, `Recommended_ROP`,
    `stockout_count_last_month` — sort by `ROP_Deficit` descending (add
    `ROP_Deficit` to the table temporarily to sort by it, then hide the
    column via the field well if you don't want it visible).

22. **Page 3 — Zone 2: Overstock.** Bar chart: axis = `Dim_Category[CategoryName]`,
    value = `Total Excess Daily Cost`. Cards: `% Holding Cost In Excess`,
    `Safe To Cut Daily Savings`, `Safe To Cut Annualised`. Table filtered
    (visual-level filter: `Safe_To_Cut` = "Safe to cut") on `item_id`,
    `category`, `stock_level`, `Target_Stock`, `Excess_Units`, `turnover_ratio`,
    sorted by `Excess_Daily_Cost` descending.

23. **Page 4 — Zone 3: Slotting.** Bar chart: axis = `Dim_Zone[ZoneName]`,
    value = `Avg Pick Time` — right-click the Y-axis → **Axis options**, keep
    the minimum at 0 (don't let Power BI auto-truncate; the near-equal bars
    *are* the finding). Second bar chart: `Dim_Zone[ZoneName]` vs
    `% Top20 In Zone` — in the Format pane, add a **Constant line** at 0.25
    labeled "expected if random." Scatter chart: X = `item_popularity_score`,
    Y = `picking_time_seconds` — leave it unaggregated (Analytics pane → add a
    trend line to visually confirm it's flat).

24. **Page 5 — Data Model.** Go back to Model view, arrange the tables neatly,
    take a screenshot (Win+Shift+S), then on a new report page **Insert →
    Image** and drop it in. Add a text box briefly explaining the grain of
    each table.

## Phase 7 — Formatting pass (20 min)

25. **View → Themes → Browse for themes**, and apply a theme (or build a
    custom one — Format pane on any visual has a paintbrush icon → check
    colors are consistent across all five pages). Keep one accent color for
    emphasis and a neutral palette otherwise — the same restraint that makes
    any dashboard easy to scan.
26. Add page titles as text boxes (not just tab names), and check every
    card/chart has a clear title — don't leave Power BI's auto-generated
    field names as chart titles.

## Phase 8 — QA: verify your build against known totals (10 min)

Before calling it done, check these on your Page 1/2/3/4 — if they're off,
a relationship or DAX formula upstream is wrong:

| Measure | Expected (approx.) |
|---|---|
| `[Total SKUs]` | 3,204 |
| `[% SKUs With Stockout]` | ~90.8% |
| `[Total Daily Holding Cost]` | ~£881,546 |
| `[Avg Pick Time]` | ~95.6 seconds |
| `[Corr Popularity vs Pick Time]` | ~0.008 |
| `[% SKUs Below Recommended ROP]` | ~97% |

## Phase 9 — Publish / share for recruiters (15-20 min)

Pick based on who's viewing and whether they have Power BI:

- **Power BI Service (best if they'll click a link).** `Home → Publish`,
  sign in with a free Microsoft account, pick "My workspace." Then in the
  service, **File → Publish to web** if you want a fully public, no-login
  link (fine here since the data is a synthetic dataset, not confidential) —
  or use **Share** for a login-gated link if you'd rather control access.
- **PDF export (best for email/CV attachment).** `File → Export → Export to
  PDF` — gives a static but universally-viewable version.
- **Portfolio repo (best overall for a job search).** Put the `.pbix`, the
  exported PDF, and a couple of page screenshots in a GitHub repo. Write your
  own short README in your own words — what the dataset was, what you built,
  what you'd do differently — since that write-up is exactly what you'll be
  asked to expand on live if it comes up in an interview. Happy to review a
  draft of that README if you want a second pair of eyes on it.
