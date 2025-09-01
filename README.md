

---

# Bahrain Imports — 2021 vs 2024 (HS Categories, Price vs Volume)

**Goal:** Identify which **HS categories** drove the change in Bahrain’s imports **between 2021 and 2024**, and split that change into **price-driven vs volume-driven** effects.

> **Key results (from the report):** 2024 imports ≈ **11.74bn** (selected currency), about **+120.9% vs 2021**. Top contributors: **Animal & Animal Products**, **Chemicals & Allied Industries**, **Vegetable Products**, **Transportation**, **Foodstuffs**, **Machinery/Electrical**. Volume-led categories include **Machinery/Electrical**, **Vegetable Products**, **Transportation**; price-led includes **Chemicals & Allied Industries**; **Animal & Animal Products** are mixed.&#x20;



---

## 🧰 Requirements

* **Power BI Desktop** (current version)
* Data files in `data/` folder (same names as above)

---

## 🗂 Data & Modeling

**Fact (appended):** `Append1`
Columns: `Date`, `Commodity No` (HS code), `Country Name`, `Import Value (BD)`, `Import Value (USA $)`, `Import Weight (KG)`, `Import Quantity`, `UM`.

**Dimensions:**

* `DimDate` (generated in DAX; marked as Date table)
* `DimHS` derived from `HS Code Category (1).csv`

  * Create **HS3** key (first 3 digits of HS code) and **deduplicate** on HS3.
  * Join **`DimHS[HS3] (1) → Append1[HS3] (*)`** (single direction).
* (Inline) `Country Name` from fact (or break out as a dim if preferred).

**Relationships (single direction):**

* `DimDate[Date] (1) → Append1[Date] (*)`
* `DimHS[HS3] (1) → Append1[HS3] (*)`

> Tip: Keep yearly staging queries **not loaded** to the model to avoid ambiguous paths.

---

## 🧠 Measures (DAX)

> Put these in a measures table or under `Append1`.

```DAX
-- Base totals
Total Import BD  = SUM ( Append1[Import Value (BD)] )
Total Import USD = SUM ( Append1[Import Value (USA $)] )
Total Weight KG  = SUM ( Append1[Import Weight (KG)] )

-- Lock comparison to 2021 vs 2024 (ignore existing Year filters)
Import 2021 =
CALCULATE ( [Total Import BD],
    REMOVEFILTERS ( DimDate[Year] ),
    KEEPFILTERS ( DimDate[Year] = 2021 )
)

Import 2024 =
CALCULATE ( [Total Import BD],
    REMOVEFILTERS ( DimDate[Year] ),
    KEEPFILTERS ( DimDate[Year] = 2024 )
)

Weight 2021 =
CALCULATE ( [Total Weight KG],
    REMOVEFILTERS ( DimDate[Year] ),
    KEEPFILTERS ( DimDate[Year] = 2021 )
)

Weight 2024 =
CALCULATE ( [Total Weight KG],
    REMOVEFILTERS ( DimDate[Year] ),
    KEEPFILTERS ( DimDate[Year] = 2024 )
)

-- KPIs
Δ Import (24–21) = [Import 2024] - [Import 2021]
% Change (24 vs 21) =
VAR Base = [Import 2021]
RETURN IF ( NOT ISBLANK(Base) && Base <> 0, DIVIDE( [Δ Import (24–21)], Base ) )

-- Price vs Volume (midpoint method)
Price 2021 = DIVIDE ( [Import 2021], [Weight 2021] )
Price 2024 = DIVIDE ( [Import 2024], [Weight 2024] )
Avg Price  = DIVIDE ( [Price 2021] + [Price 2024], 2 )
Avg Qty    = DIVIDE ( [Weight 2021] + [Weight 2024], 2 )

Volume Effect (24–21) = ( [Weight 2024] - [Weight 2021] ) * [Avg Price]
Price  Effect  (24–21) = ( [Price 2024]  - [Price 2021]  ) * [Avg Qty]

-- Optional rounding check (≈0)
Check: Δ vs (P+V) = [Δ Import (24–21)] - ( [Price  Effect (24–21)] + [Volume Effect (24–21)] )

-- Ranking helpers
Category Rank by Δ =
RANKX ( ALL('HS Code Category (1)'[Category]), [Δ Import (24–21)], , DESC, DENSE )
Country Rank by 2024 =
RANKX ( ALL(Append1[Country Name]), [Import 2024], , DESC, DENSE )
```

---

## 📈 Report Pages & Visuals

1. **KPI Cards**

   * `% Change (24 vs 21)`
   * `Import 2024`
   * `Δ Import (24–21)`

2. **Categories Driving Change (2021→2024)**

   * *Clustered Bar*: Axis = HS Category; Value = `Δ Import (24–21)`
   * Sort by `Δ Import (24–21)` (desc); optional Top N = 10.

3. **Price vs Volume Effects (2021→2024)**

   * *Clustered Bar*: Values = `Volume Effect (24–21)`, `Price Effect (24–21)`
   * Optional **100% stacked** variant to show shares.

4. **Top Supplier Countries — Top-5 Categories (2021 vs 2024)** *(optional compact panel)*

   * *Small multiples* column chart
   * Axis = Country; Values = `Import 2021`, `Import 2024`
   * Small multiples = Category; Filters: Top-5 by `Δ Import (24–21)`, and Top-5 countries by `Import 2024`.

---

## ▶️ How to Run

1. Open **`Power Bi Project.pbix`** in Power BI Desktop.
2. Ensure the `data/` files are present and paths resolve (Power Query).
3. Refresh.
4. The main page presents 2021→2024 comparison by default.
5. Use the currency toggle (if enabled) to switch **BD / USD**.

---

## 🔬 Methodology Notes

* **HS mapping** at **HS3** level (first 3 digits) to group commodities into categories.
* **Price vs Volume** uses a **midpoint** (Shapley-style) decomposition to attribute change fairly between price and quantity.
* **Single-direction** relationships and a **DimDate** table are used to keep time intelligence stable.

---

## 🚧 Known Limitations / Next Steps

* Mapping at HS3 is broad; HS4/HS6 would yield more granular insights if classifications are available.
* Add a **field parameter** to switch metrics (BD vs USD) across all visuals.
* Consider **drillthrough** pages for each category with country trends.

---

## 📜 Attribution

* Visuals and figures summarized here originate from the exported **Power BI** report included in this repository.&#x20;

---

If you want, I can also save this as a `README.md` file and attach it here for download.
