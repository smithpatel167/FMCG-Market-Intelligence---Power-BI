# 📊 FMCG Market Intelligence — Power BI Portfolio Project

A end-to-end consumer panel analytics report built in Power BI, simulating the kind of market intelligence work done at insights firms.

---
## 🔗 Live Report
[View Interactive Report on Power BI Service](https://app.powerbi.com/view?r=eyJrIjoiMDM3ZDlkYWMtODE2ZS00MTM5LTllOTAtNmI0NTEyMDRhNmVkIiwidCI6ImMxNjQxMjEyLWJkODItNDg1Ny1hMTE1LWQxYjdiY2I4NWFjNSJ9)
## 🗂️ Project Overview

This project demonstrates the full analytics workflow:
- **Raw messy data** → Power Query cleaning → **Star schema data model**
- Custom **DAX measures** for consumer panel KPIs
- **4-page interactive Power BI report** with cross-page slicer sync, drill-through, bookmarks, and field parameters

**Dataset:** 100% synthetic — randomly generated for portfolio purposes. All brand names, market figures, and dynamics are fictional.

---

## 📁 Repository Structure

```
📦 FMCG-Market-Intelligence-PowerBI
 ┣ 📂 dataset
 ┃ ┗ 📄 FMCG_Market_RAW_Export.xlsx     ← Raw messy input (3 annual sheets)
 ┣ 📂 report
 ┃ ┗ 📄 FMCG_Market_Intelligence.pbix   ← Power BI report file
 ┣ 📂 screenshots
 ┃ ┣ 🖼️ page1_executive_overview.png
 ┃ ┣ 🖼️ page2_brand_battleground.png
 ┃ ┣ 🖼️ page3_geographic_intelligence.png
 ┃ ┗ 🖼️ page4_shopper_behavior.png
 ┗ 📄 README.md
```

---

## 🧰 Tools & Technologies

| Tool | Usage |
|---|---|
| **Power BI Desktop** | Report building, data modeling, DAX |
| **Power Query (M)** | Data cleaning, transformation, star schema build |
| **DAX** | KPI measures, time intelligence, market share calculations |
| **Excel** | Raw data source (synthetic) |

---

## 📦 Dataset Details

| Attribute | Details |
|---|---|
| **Categories** | Carbonated Soft Drinks, Salty Snacks, Energy Drinks, Personal Care, Household Cleaning |
| **Brands** | 27 fictional brands (5–6 per category, including Private Label) |
| **Channels** | Grocery, Mass, Dollar/Discount, eCommerce, Club, Drug, Convenience |
| **Geography** | All 50 US States |
| **Time Period** | Jan 2023 – Dec 2025 (36 months) |
| **Rows** | ~216,000 fact rows |
| **Grain** | Month × State × Channel × Brand |
| **Metrics** | Units, Revenue, Promo Units, Promo Revenue, Buying HH, Repeat HH, New HH, Shopping Trips, Avg Selling Price |

### Raw File — Intentional Mess (Power Query Practice)
The raw Excel file mimics a real-world panel data export dump:
- 3 sheets (one per year) with **different date formats** (`Mmm-YY`, `MM/YYYY`, `YYYY-MM`)
- 4 title/metadata rows above the real header on each sheet
- Case and whitespace noise on all categorical columns
- Promo Units + Revenue **combined in one text column**
- Revenue and Units stored as **formatted text** (`$1,234.56`, `"N/A"` for ~2.5% of rows)
- ~1% exact duplicate rows per sheet
- Blank rows scattered throughout
- Grand Total row at the bottom of each sheet
- Brand_Info sheet with missing Price Tier for some brands

---

## 🔄 Power Query — Data Cleaning Steps

Each of the 3 raw sheets was cleaned independently in Power Query, then appended:

1. Remove 4 junk header rows
2. Promote first row as headers
3. Rename columns (strip stray spaces)
4. Filter out blank rows + Grand Total row (single filter on `StateName`)
5. Remove duplicate rows
6. Trim + Clean + Capitalize Each Word on all text columns
7. Parse date from text (different M formula per sheet due to format differences)
8. Replace `"N/A"` with null → strip `$` and `,` → change to numeric type
9. Split combined Promo column (`"142 units / $1,234.56"`) into two separate columns
10. Fill null HH split columns with 0

After cleaning and appending, dimension tables were extracted via **Reference queries**:
- `Dim_Date` (with calculated Year, Quarter, Month columns) - I just created seprate date table using DAX.
- `Dim_Channel`
- `Dim_Category`

`Dim_Brand` and `Dim_Geography` were loaded from the Brand_Info and State_Reference sheets respectively.

---

## 🗃️ Data Model — Star Schema

```
                    Dim_Date
                       │
     Dim_Category ─────┤
                       │
     Dim_Brand ────────┼──── Fact_Sales ────┬──── Dim_Channel
                       │                    │
     Dim_Geography ────┘                    └──── (all measures
                                                   live in _Measures table)
```

All relationships: **one-to-many, single direction**, Dim → Fact.

---

## 📐 Key DAX Measures

### Market Share
```dax
Category Revenue =
CALCULATE([Total Revenue], ALL(Dim_Brand))

Market Share % =
DIVIDE([Total Revenue], [Category Revenue])
```
`ALL(Dim_Brand)` removes the brand filter from context while preserving all other active filters (date, category, channel) — so the denominator always reflects the full category.

### Time Intelligence
```dax
Revenue PY =
CALCULATE([Total Revenue], SAMEPERIODLASTYEAR(Dim_Date[Date]))

Revenue YoY % =
DIVIDE([Total Revenue] - [Revenue PY], [Revenue PY])
```

### Promo Dependency
```dax
Promo Revenue % = DIVIDE([Promo Revenue], [Total Revenue])
Base Revenue    = [Total Revenue] - [Promo Revenue]
```

### Shopper Metrics
```dax
Repeat Rate %   = DIVIDE(SUM(Fact_Sales[RepeatBuyingHouseholds]), [Total Buying HH])
Trips per Buyer = DIVIDE([Total Trips], [Total Buying HH])
Units per Trip  = DIVIDE([Total Units], [Total Trips])

-- Volume decomposition identity:
-- Total Units = Buying HH × Trips per Buyer × Units per Trip
```

### Rolling Average
```dax
Revenue 3M Rolling Avg =
AVERAGEX(
    DATESINPERIOD(Dim_Date[Date], MAX(Dim_Date[Date]), -3, MONTH),
    CALCULATE([Total Revenue])
)
```

### Dynamic Top N
```dax
Brand Revenue Rank =
RANKX(ALL(Dim_Brand[BrandName]), [Total Revenue], , DESC, DENSE)

Show In Top N =
IF([Brand Revenue Rank] <= [TopN Parameter Value], 1, 0)
```

---

## 📊 Report Pages

### Page 1 — Executive Overview
**Story:** How big is the category, how fast is it growing, and which brands are winning?

- 4 KPI cards: Total Revenue, Total Units, Total Buying HH, Promo Revenue %
- Category Revenue trend (multi-line chart — 3-year view)
- Revenue YoY Change waterfall (brand level, Energy Drinks focus)
- Top N brand bar chart (dynamic slicer: Top 5 / 10 / All)
- Category revenue donut
- Year and Category slicers

### Page 2 — Brand Battleground
**Story:** Who's gaining share, who's losing it, and are they buying share with promo spend or earning it organically?

- Market Share % ranked bar with Share Change pp labels
- Share Trend small multiples (one line per brand over 3 years)
- Promo Dependency vs Market Position scatter
- Energy Drinks brand revenue line chart (NovaSurge rise vs IronEdge decline)
- Category slicer

### Page 3 — Geographic Intelligence
**Story:** Which states and regions are driving growth, and where does each brand over/under-index?

- Shape Map (US choropleth) with Field Parameter metric switching: Revenue / HH Penetration % / Revenue YoY % / Market Share %
- Revenue Efficiency by Region bar (Revenue per HH — normalised for state size)
- Top 10 States snapshot
- West Revenue Share card
- EcoShield Spotlight bookmark button
- Brand, Year, Channel slicers

### Page 4 — Shopper Behavior & Channel Mix
**Story:** Who is buying, how often, and through which channels?

- Buyer Base stacked bar — Loyalists vs Trialists per brand
- Channel Revenue Mix 100% stacked bar per brand
- Channel × Category Performance matrix (with data bars + conditional formatting)
- Shopping Frequency Trend — Trips per Buyer (with 3M rolling avg line)
- Price Tier, Year, Category slicers

---

## 💡 Built-in Dataset Storylines

These dynamics are baked into the synthetic data — each one is a genuine analytical talking point:

| Storyline | Where to see it |
|---|---|
| **NovaSurge** (Energy Drinks challenger) growing 16%/yr, taking share from IronEdge | Page 2 — line chart + small multiples |
| **Private Label** gaining share in CSD and Household Cleaning | Page 2 — market share bar, 2023 vs 2025 |
| **ThunderVolt** has 45% promo dependency — flat base sales despite high share | Page 2 — promo dependency scatter |
| **EcoShield** (sustainable cleaning) over-indexes heavily in West Coast states | Page 3 — map + EcoShield Spotlight bookmark |
| **PureRoot** Personal Care drives disproportionate eCommerce revenue | Page 4 — channel mix 100% bar |
| **Zenith Zero** (zero-sugar CSD) growing 12%/yr on health trend | Page 1 — category trend line |
| **Average selling price** drifts up ~3.8%/yr across all categories | Page 1 — KPI cards over time |

---

## 🚀 How to Use

1. Download `FMCG_Market_RAW_Export.xlsx` and `FMCG_Market_Intelligence.pbix`
2. Open the `.pbix` file in **Power BI Desktop**
3. If prompted to update data source, point it to the downloaded Excel file
4. Use the **Year** and **Category** slicers (synced across pages) to explore
5. On Page 3 — use the **Map Metric** slicer to switch the map between Revenue / Penetration / YoY% / Market Share

---


## 👤 About

Built by **Smith Patel** — FMCG & Consumer Insights Analytics | Power BI | DAX | Power Query

[LinkedIn](https://www.linkedin.com/in/smithpatel167/) • [GitHub](https://github.com/smithpatel167)
