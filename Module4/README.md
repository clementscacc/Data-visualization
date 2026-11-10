# Module 4 - Data Set Repair

**Clement Scaccia, Data Visualization**

For this module I picked 2 of my 4 data sets from Module 3 and fixed the problems I found in them with Python (pandas) in a Jupyter notebook.

| File | What it is |
|---|---|
| [`Module4_Data_Repair.ipynb`](Module4_Data_Repair.ipynb) | Notebook that finds each problem, fixes it, and checks the fix |
| [`cleaned_data/us_cpi_monthly_bls_cleaned.csv`](cleaned_data/us_cpi_monthly_bls_cleaned.csv) | Repaired CPI data |
| [`cleaned_data/cboe_vix_daily_cleaned.csv`](cleaned_data/cboe_vix_daily_cleaned.csv) | Repaired VIX data |

The original files in [`../data`](../data) were not changed. Every fix is flagged with a column in the cleaned files, so you can always tell which values are original and which were repaired.

---

## Data set 1: US Consumer Price Index (Government)

Source: U.S. Bureau of Labor Statistics, CPI-U, monthly since 1913 ([datasets/cpi-us](https://github.com/datasets/cpi-us)). Raw file: `data/government/us_cpi_monthly_bls.csv`

### Failure 1: October 2025 is missing

- **What was wrong:** the data is supposed to have one row per month, but October 2025 isn't there. 1,365 months in the date range, only 1,364 rows.
- **Why:** BLS couldn't collect prices during the fall 2025 government shutdown, so it never published October 2025 CPI ([BLS FAQ](https://www.bls.gov/cpi/additional-resources/2025-federal-government-shutdown-impact-cpi-faq.htm)).
- **Why it matters:** charts connect September straight to November like nothing's missing, and any calculation that counts rows as months (like "12 rows back = 1 year ago") is off by one month for every date after that.
- **How I fixed it:** I put the month back and estimated the Index with linear interpolation (halfway between September and November). I added an `Estimated` column (True only for Oct 2025) so it's clear it's not an official number.

### Failure 2: the `Inflation` column is misleading, and November 2025 is wrong

- **What was wrong:**
  1. The column is the **month over month** % change, but the name makes it look like the normal yearly inflation rate. For example, Sep 2026 shows 0.59, while the actual 12-month inflation is about 4%.
  2. Because October was missing, the November 2025 value (-0.21%) was calculated against September, so it's really a **2-month** change.
- **How I fixed it:** I removed the old column and recalculated it from the repaired Index as `MoM_Change_%`, so every row is a real 1-month change. I also added `YoY_Inflation_%` (12-month change), which is the inflation number people actually use, and a `MoM_Uses_Estimate` flag for Oct and Nov 2025 since they depend on the estimated value.

---

## Data set 2: Cboe VIX Volatility Index (Commercial)

Source: Cboe Global Markets, daily Open/High/Low/Close since 1990 ([datasets/finance-vix](https://github.com/datasets/finance-vix)). Raw file: `data/commercial/cboe_vix_daily.csv`

### Failure 1: fake Open/High/Low values in 1990-1991

- **What was wrong:** 505 rows (all of 1990 and 1991) have Open = High = Low = Close, so it looks like the VIX didn't move at all for 2 years. Cboe only recorded the close back then, and the other columns were just copies of it. The dataset's notes say this lasts until June 2004, but my check showed it actually stops at the end of 1991.
- **Why it matters:** the average daily range (High - Low) for 1990-91 comes out as **0.00**, compared to about **1.11** for 1992-95. Candlestick or range charts would be wrong.
- **How I fixed it:** I set Open, High and Low to empty for those rows, kept the real Close, and added a `Close_Only` flag column.

### Failure 2: 47 days where the Open is outside the High-Low range

- **What was wrong:** on 47 days (mostly 1992-1996) the Open is higher than the High or lower than the Low, which is impossible. The Close is always inside the range, so only the Open is bad.
- **How I fixed it:** I don't know the real opening value, so instead of making up a number I set those 47 Opens to empty and added an `Open_Invalid` flag column.

### Failure 3: a row for a day the market was closed

- **What was wrong:** there's a row for June 11, 2004 with O = H = L = C = 15.04 (the same as the previous close). Markets, Cboe included, were closed that day for President Reagan's national day of mourning ([SEC notice](https://www.sec.gov/news/press/2004-77.htm)).
- **How I fixed it:** I removed the row so it doesn't count as a fake "no change" trading day. Rows went from 9,278 to 9,277.

**Checked but not a failure:** the gaps around 9/11/2001, Jan 2, 2007 (President Ford's funeral) and Oct 29-30, 2012 (Hurricane Sandy) are real market closures, so I left them alone.

---

## How the fixes are checked

At the end of each section the notebook runs `assert` checks, and they all pass:
- CPI: every month from 1913 to 2026 is there, no missing Index values, exactly 1 estimated value
- VIX: Low ≤ Open ≤ High and Low ≤ Close ≤ High on every row that has those values, no duplicate dates, no row for June 11, 2004
