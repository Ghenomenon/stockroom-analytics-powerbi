# From Raw Data to Actionable Insight — A Case Study

Companion piece to the main Stockroom Analytics report. That project shows the
output of a rigorous analysis; this one shows the *process* — using the
[Diamonds Characteristics and Pricing Analysis dataset](https://www.kaggle.com/datasets/muhammadahmaddaar/diamonds-characteristics-and-pricing-analysis)
on Kaggle, a repost of the classic 53,940-row diamonds dataset (`carat`,
`cut`, `color`, `clarity`, `depth`, `table`, `price`, `x`, `y`, `z`).

## The four-step framework

Every finding in the Stockroom report follows the same sequence. Skipping any
one of these steps is how dashboards end up full of numbers nobody trusts.

1. **Frame a business question.** Not "what does the data show" but "what
   decision does this change." Stockroom's version: *are reorder points set
   correctly?* Diamonds' version: *does a better cut command a higher price?*
2. **Build a clean, well-typed data model.** Grain, keys, and known data
   quality issues documented before any chart gets built.
3. **Compute the obvious statistic — then interrogate it before you trust
   it.** Group averages, correlations, and trend lines are cheap to produce
   and easy to misread. Check them against a second method before they go in
   front of a decision-maker.
4. **Convert the statistic into a specific, actionable recommendation.** Not
   "cut matters" but "reprice these 140 SKUs" or "these three cut grades are
   underpriced relative to comparable carat weight."

The diamonds dataset is a good teaching example because step 3 is where a
naive analysis actively lies to you.

## Step 1–2: question and data model

**Question:** does diamond cut quality (Fair → Good → Very Good → Premium →
Ideal) predict price, the way a jeweller pricing inventory would expect?

**Data model:** one row per stone, grain confirmed unique, no join needed —
this is a flat fact table, unlike Stockroom's star schema. The columns that
matter for this question: `cut` (5-level ordinal), `carat` (continuous,
right-skewed, range roughly 0.2–5.01), `price` (right-skewed, roughly
$326–$18,823). A known data quality issue in this dataset: a small number of
rows have `x`, `y`, or `z` recorded as 0mm, which is physically impossible for
a cut stone — those rows get flagged and excluded from any calculation that
uses physical dimensions.

## Step 3: the naive statistic, and why it's wrong

Group by cut, average the price. This is the first thing almost anyone runs:

| Cut | Avg. price | Avg. carat |
|---|---|---|
| Fair | ~$4,358 | ~1.05 |
| Good | ~$3,929 | ~0.85 |
| Very Good | ~$3,982 | ~0.81 |
| Premium | ~$4,584 | ~0.89 |
| Ideal | ~$3,458 | ~0.70 |

Read on its own, that table says **Fair-cut stones cost more than Ideal-cut
stones.** Taken at face value, a jeweller using this to audit pricing would
conclude their best-cut inventory is systematically underpriced and their
worst-cut inventory is overpriced — the opposite of reality, and a
recommendation that would actively lose money if acted on.

The `Avg. carat` column is the tell. It's doing all the work: Fair-cut
stones average roughly 50% more carat weight than Ideal-cut stones. Carat is
the single biggest driver of price — larger rough stones are harder to cut
into ideal proportions without sacrificing weight, so cutters more often
finish a large stone as Fair or Good rather than cut it down to Ideal
proportions. Cut quality and carat are confounded. The naive average isn't
measuring "does cut affect price," it's measuring "cut correlates with
carat, and carat drives price" — a completely different fact wearing the
first one's clothes.

This is exactly the failure mode the Stockroom report guards against with
cross-checked correlations rather than single-pass aggregates (see its
`README.md`, "Notable technique: correlation in DAX"). A number that isn't
interrogated before it's presented isn't an insight, it's a liability.

## Step 3 continued: controlling for the confound

Two fixes, both simple:

- **Price per carat** instead of raw price, computed per stone before
  averaging by cut.
- **Regress `log(price)` on `log(carat)` plus cut/color/clarity as
  categorical terms**, so cut's coefficient is the price effect *holding
  carat constant*. Log-log because both price and carat are right-skewed and
  the relationship between them is closer to a power law than a straight
  line.

Either way, the sign flips. Once carat is held constant, Ideal cut carries a
real, positive premium over Fair — on the order of 10–15% for stones of
equivalent carat, color, and clarity. The relationship the naive table
denied is the one that was actually there.

## Step 4: the actionable recommendation

"Cut affects price, once you control for carat" is still not an action. The
version that is:

- Fit `log(price) ~ log(carat) + cut + color + clarity` on the full dataset.
  That model's prediction is a defensible fair-market price for any stone
  given its four Cs.
- Score every stone in current inventory against that prediction. Flag
  stones priced more than ~15% below the model's estimate as **underpriced —
  reprice up**, and more than ~15% above as **overpriced — reprice down or
  expect it to sit**.
- This is the same move as Stockroom's `Safe_To_Cut` flag: a statistical
  relationship, cross-checked, turned into a row-level, dollar-specific
  action list rather than left as a chart.

## Reproduce it

The exact figures above are the well-documented reference statistics for the
classic diamonds dataset; this sandbox has no outbound access to Kaggle to
re-pull this specific repost, so treat them as expected values to confirm
against your own copy of the CSV, not as numbers pulled fresh for this
write-up. Run this once you have the file locally:

```python
import pandas as pd
import numpy as np

df = pd.read_csv("diamonds.csv")
df = df[(df.x > 0) & (df.y > 0) & (df.z > 0)]  # drop impossible dimensions

# the naive (misleading) view
naive = df.groupby("cut")[["price", "carat"]].mean()
print(naive)

# controlling for carat via price-per-carat
df["ppc"] = df.price / df.carat
print(df.groupby("cut")["ppc"].mean())

# controlling for carat via log-log regression
import statsmodels.formula.api as smf
model = smf.ols(
    "np.log(price) ~ np.log(carat) + C(cut) + C(color) + C(clarity)",
    data=df,
).fit()
print(model.summary())
```

## Takeaway

The two case studies in this repo point at the same principle from different
industries:

| Principle | Stockroom example | Diamonds example |
|---|---|---|
| A naive aggregate can point the wrong way | — | "Fair cut costs more than Ideal" |
| The fix is to isolate the confound | Correlation checked, not assumed | Control for carat before crediting cut |
| Insight only counts once it's a specific action | `Safe_To_Cut` SKU list with £ savings | Per-stone reprice flag against model price |

Turning data into insight isn't running an aggregation and screenshotting the
result. It's asking what else could produce that number, checking, and only
then writing down what to do about it.
