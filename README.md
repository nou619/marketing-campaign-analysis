# Skincare Marketing Campaign Analysis
### Data Cleaning & Preparation Pipeline

---

##  Project Overview

This project cleans and prepares a raw skincare marketing campaigns dataset (`skincare_marketing_campaigns.csv`) for downstream analysis and modelling. The notebook handles messy real-world data issues — mixed date formats, missing values, inconsistent casing, duplicate rows, and funnel integrity violations — producing a fully validated output file ready for EDA, dashboards, or ML pipelines.

**Input:** `skincare_marketing_campaigns.csv`  
**Output:** `clean_marketing_data.csv`

---

##  Repository Structure

```
├── Untitled.ipynb                      # Main data cleaning notebook
├── skincare_marketing_campaigns.csv    # Raw input data
├── clean_marketing_data.csv            # Cleaned output data
├── Skincare_Campaign_Analysis.pptx     # Presentation summary
└── README.md
```

---

##  Dataset Schema

| Column | Type | Description |
|---|---|---|
| `campaign_id` | String | Unique campaign identifier |
| `campaign_name` | String | Marketing campaign name |
| `target_audience` | String | Audience segment targeted |
| `channel` | String | Marketing channel (social_media, email, influencer, etc.) |
| `date` | DateTime | Campaign date |
| `spend` | Float | Budget spent ($) |
| `views` | Int | Impressions / views |
| `clicks` | Int | Number of clicks |
| `conversions` | Int | Number of conversions |
| `revenue` | Float | Attributed revenue ($) |

---

##  Cleaning Steps

### 1. Whitespace Stripping
Trimmed leading/trailing whitespace from all string columns: `campaign_id`, `campaign_name`, `target_audience`, `channel`.

```python
df['campaign_name'] = df['campaign_name'].str.strip()
```

### 2. Case Normalisation
Lowercased the `channel` column to prevent double-counting due to casing inconsistencies (e.g. `"Social_Media"` vs `"social_media"`).

```python
df['channel'] = df['channel'].str.lower()
```

### 3. Date Parsing
The `date` column contained 5+ distinct formats with mixed separators. Solved with a two-step regex normalisation + pandas mixed-format parser:

```python
df['date'] = df['date'].str.replace(r"\D+", "-", regex=True)
df['cleaned_date'] = pd.to_datetime(df['date'], format="mixed", errors="coerce")
```

Zero `NaT` values after parsing — no dates were lost.

### 4. Duplicate Removal
Detected and dropped exact duplicate rows.

```python
df = df.drop_duplicates()
```

### 5. Negative Value Validation
Verified that all numeric columns (`spend`, `views`, `clicks`, `conversions`, `revenue`) contained no negative values.

### 6. Missing Value Imputation

Missing values were handled differently per column based on the nature of each metric:

| Column | Strategy | Rationale |
|---|---|---|
| `spend` | Median per `[channel × campaign_name]` | Spend varies widely by channel; zero cannot be assumed |
| `views` | Median per `[channel × campaign_name]` | Context-aware imputation preserves distribution shape |
| `clicks` | Median per `[channel × campaign_name]`, then structural zeros applied | Rows with `conversions = 0` forced to `clicks = 0` |
| `conversions` | Median per `[channel × campaign_name]` | Same group-median approach |
| `revenue` | **Left as NaN** (partially) | Imputing outcome metrics would fabricate business results; analysis on revenue uses only real values |

**Structural zero enforcement:**
```python
df.loc[df["conversions"] == 0, "clicks"] = 0
df.loc[df["conversions"] == 0, "revenue"] = 0
```

**Revenue imputation (for remaining gaps via RPC):**
```python
df['rpc'] = df['revenue'] / df['clicks'].replace(0, np.nan)
df['rpc'] = df.groupby(['channel','campaign_name'])['rpc'].transform(lambda x: x.fillna(x.median()))
df['revenue'] = df['revenue'].fillna((df['rpc'] * df['clicks']).round()).clip(lower=0)
```

### 7. Missing Flags
Boolean flag columns were retained in the output for audit trails and downstream sensitivity analysis:

- `views_was_missing`
- `spend_was_missing`
- `conversions_was_missing`
- `clicks_missing` / `clicks_imputed_flag`
- `revenue_was_missing`

---

##  Key Findings

| Metric | Value |
|---|---|
| Missing % — revenue (before) | ~18.5% |
| Missing % — spend (before) | ~12.3% |
| Duplicate rows removed | Yes |
| Date formats handled | 5+ |
| Final data completeness | ~96% (revenue NaN intentionally retained where present) |
| All numeric negatives found | None |

---

##  Design Decisions

**Why group median over global median?**  
Different channels operate at radically different scale. A podcast campaign's typical spend is nothing like a social media campaign's. Imputing with `groupby(['channel', 'campaign_name']).median()` keeps imputed values within a realistic range for each context.

**Why leave some revenue as NaN?**  
Revenue is the primary outcome metric. Fabricating it — even via a reasonable proxy like RPC — risks corrupting ROI and ROAS calculations. Where RPC-based imputation was applied, it is flagged via `revenue_was_missing` so analysts can exclude those rows for conservative analysis.

**Why enforce structural zeros?**  
The marketing funnel is causal: you cannot have conversions without clicks, or revenue without conversions. Imputing non-zero clicks onto zero-conversion rows would violate this logic and inflate funnel metrics.

---

##  How to Run

1. Clone the repo and place `skincare_marketing_campaigns.csv` in the root directory.
2. Install dependencies:
   ```bash
   pip install pandas numpy
   ```
3. Open and run `Untitled.ipynb` top to bottom.
4. Output saved to `clean_marketing_data.csv`.

---

##  Next Steps

- **EDA** — Explore distributions, channel ROI, and spend vs revenue correlations
- **Modelling** — Predict conversions or optimise budget allocation across channels
- **Dashboard** — Build live reporting from `clean_marketing_data.csv`
- **Sensitivity analysis** — Re-run key metrics with imputed rows excluded to validate robustness

---

##  Dependencies

| Package | Version |
|---|---|
| pandas | ≥ 1.5 |
| numpy | ≥ 1.23 |

---
