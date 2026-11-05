# North Austin Housing – XGBoost Dataset

Monthly dataset for building an XGBoost model of Austin-area home prices. One row per month, **Jul 2016 – Sep 2026 (123 rows)**, with macro, local-economy and housing-market columns plus ready-made time features (calendar fields, lags, rolling averages).

File: `north_austin_housing_xgboost_data.xlsx`
Built: 5 Nov 2026

> **Geography caveat:** the housing columns cover the whole **Austin-Round Rock metro**, not North Austin ZIP codes (78758, 78759, etc.). ZIP-level sources could not be pulled; see [Not included](#not-included).

## Workbook tabs

| Tab | What it holds |
|---|---|
| `Model_Data` | The flat modelling table (23 columns). Inputs are typed values; features are live formulas. |
| `Data_Dictionary` | Definition and calculation for every column, a colour key, and notes on missing values. |
| `Sources_Gaps` | Source, series ID, URL and last-updated date for each series, plus what was not pulled and why. |
| `Mortgage_Weekly` | 534 weekly 30-year mortgage rates (Jul 2016 – Sep 2026). `Model_Data` averages these by month. |
| `Raw_CPI` | CPI index from Jul 2015, so year-over-year inflation is available from the first row. |

## Columns in `Model_Data`

| Group | Columns |
|---|---|
| Time | `date`, `year`, `quarter`, `month` |
| Housing market (Austin metro, Realtor.com) | `median_list_price` (candidate target), `active_listing_count`, `median_days_on_market` |
| Macro | `mortgage_rate_30y_pct`, `fed_funds_rate_pct`, `cpi_index`, `cpi_yoy_pct` |
| Local economy (Austin MSA, BLS) | `unemployment_rate_pct`, `nonfarm_employment_k`, `information_employment_k` |
| Lag features | `price_lag_1`, `price_lag_2`, `price_lag_3`, `inventory_lag_1`, `inventory_lag_3`, `dom_lag_1` |
| Rolling features | `dom_roll3`, `inventory_roll3`, `price_roll3_lagged` |

Notes on specific columns:

- `mortgage_rate_30y_pct` is the average of the weekly Freddie Mac rates for weeks ending in that month.
- `cpi_yoy_pct` is CPI divided by CPI 12 months earlier, minus 1, in percent.
- `information_employment_k` is a rough proxy for tech jobs. The information sector is broader than tech and misses software jobs classed under professional services.
- `price_roll3_lagged` averages the **prior** three months (T-3 to T-1) and excludes the current month, so it cannot leak the target.
- `dom_roll3` and `inventory_roll3` include the current month (T-2 to T).

## Data sources

| Data | Series | Publisher |
|---|---|---|
| 30-year mortgage rate (weekly) | PMMS | [Freddie Mac](https://www.freddiemac.com/pmms/archive) (also FRED `MORTGAGE30US`) |
| Federal funds rate | [`FEDFUNDS`](https://fred.stlouisfed.org/series/FEDFUNDS) | FRED |
| CPI-U, all items | [`CPIAUCSL`](https://fred.stlouisfed.org/series/CPIAUCSL) | FRED / BLS |
| Austin MSA unemployment rate | [`AUST448URN`](https://fred.stlouisfed.org/series/AUST448URN) | FRED / BLS |
| Austin MSA total nonfarm jobs | [`AUST448NA`](https://fred.stlouisfed.org/series/AUST448NA) | FRED / BLS |
| Austin MSA information-sector jobs | [`AUST448INFON`](https://fred.stlouisfed.org/series/AUST448INFON) | FRED / BLS |
| Median listing price | [`MEDLISPRI12420`](https://fred.stlouisfed.org/series/MEDLISPRI12420) | Realtor.com via FRED |
| Active listing count | [`ACTLISCOU12420`](https://fred.stlouisfed.org/series/ACTLISCOU12420) | Realtor.com via FRED |
| Median days on market | [`MEDDAYONMAR12420`](https://fred.stlouisfed.org/series/MEDDAYONMAR12420) | Realtor.com via FRED |

The series were read through a text-fetching tool. Month alignment, row counts and weekly spacing were checked in code, and the derived columns were verified against independent calculations. Spot-check anything critical against the links above.

## Missing values

- **Oct 2025:** `cpi_index`, `cpi_yoy_pct` and `unemployment_rate_pct` are blank because the source series have no value for that month. They are left as true blanks (NaN) rather than forward-filled, and XGBoost handles NaN natively.
- **First 1–3 rows:** lag and rolling columns are blank because no earlier history was pulled. Drop the first 3 rows before training.
- Everything else is complete for all 123 months.

## Not included

These were requested but could not be pulled. Details and steps are on the `Sources_Gaps` tab.

| Source | Why it is missing | How to get it |
|---|---|---|
| Redfin Data Center (ZIP-level) | Bulk compressed download, not retrievable from this workspace. | redfin.com/news/data-center/downloads, ZIP Code Market Tracker, filter to your ZIPs. This is the best route to real North Austin sale prices. |
| Texas A&M Real Estate Research Center | Data loads dynamically behind a geography picker. ZIP detail is limited to REALTORS members. | trerc.tamu.edu/data/housing-activity, pick the Austin-Round Rock-San Marcos MSA, export CSV. |
| Census ACS (income, population, migration) | The API requires a free key. | Request one at api.census.gov/data/key_signup.html. ACS is annual, so forward-fill to months. |
| Texas Workforce Commission LMI | Interactive portal. Equivalent BLS series from FRED were used instead. | TWC labor market information portal, for extra industries. |
| Unlock MLS | Reports are infographics and PDFs, not tables. Only about three years are online. | unlockmls.com/document-library/market-statistics, or ask support for older reports. |

## Quick start

```python
import pandas as pd

df = pd.read_excel(
    "north_austin_housing_xgboost_data.xlsx",
    sheet_name="Model_Data",
    parse_dates=["date"],
).set_index("date")

df = df.iloc[3:]  # first 3 months have incomplete lags

target = "median_list_price"
X = df.drop(columns=[target])
y = df[target]

# Split by time, never randomly
split = int(len(df) * 0.8)
X_train, X_test = X.iloc[:split], X.iloc[split:]
y_train, y_test = y.iloc[:split], y.iloc[split:]
```

Modelling cautions:

- **Nowcast vs. forecast:** the same-month market columns (`active_listing_count`, `median_days_on_market`, `dom_roll3`, `inventory_roll3`) describe the month you are predicting. Keep them only if you are nowcasting. To forecast ahead, use the lag and `price_roll3_lagged` columns and drop or shift the concurrent ones.
- **Time-based validation:** use a time-ordered split or `TimeSeriesSplit`. Random splits leak the future into training.
- **Listing vs. sale price:** `median_list_price` is what sellers ask, not what homes close for. Swap in Redfin's sale price once you have it and merge on `date`.
- **2020–2022 distortions:** the pandemic shock and the 2021–2022 housing run-up dominate the variance. Consider a regime indicator or check how the model does on 2023 onward.
