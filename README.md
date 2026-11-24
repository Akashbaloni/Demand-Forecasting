# 📦 Weekly Demand Forecasting — Stock Planner

[![CI](https://github.com/Akashbaloni/Demand-Forecasting/actions/workflows/ci.yml/badge.svg)](https://github.com/Akashbaloni/Demand-Forecasting/actions/workflows/ci.yml)

Forecast how many units of each product a UK online gift retailer will sell in each of the **next 8 weeks**, and turn that forecast into a simple stock decision: **how much stock to keep**.

🔗 **Live app:** https://demand-forecasting-o8yb.onrender.com
📄 **API docs:** https://demand-forecasting-o8yb.onrender.com/docs

> The app runs on Render's free plan. If nobody has used it for a while, the first load can take 30–60 seconds while the server wakes up.

![Stock Planner UI](assets/screenshot.png)

---

## Problem

A retailer has to decide **every week** how much of each product to keep in stock.

- **Too little stock**: customers leave and sales are lost.
- **Too much stock**: money is tied up in boxes on a shelf.

A single "best guess" number is not enough for this decision. The planner also needs to know **how high demand could reasonably go**. So this project forecasts a **range** for each week, not just one number:

| In the app | What it means | Model output |
|---|---|---|
| **Likely to sell** | The middle forecast | p50 (median) |
| **Stock to keep** | Enough for a busy week; demand should go above this only about 1 week in 10 | p90 |
| **Safety stock** | The extra on top of "likely" | p90 − p50 |

---

## Dataset

**Online Retail II**, UCI Machine Learning Repository
https://archive.ics.uci.edu/dataset/502/online+retail+ii

- Every transaction of a UK-based online gift-ware retailer from **01 Dec 2009 to 09 Dec 2011**. Many of the customers are wholesalers.
- About 1.07 million rows in one Excel file (`online_retail_II.xlsx`, 45.6 MB) with two sheets, one per year.
- Licence: **CC BY 4.0**.

> Chen, D. (2012). *Online Retail II* [Dataset]. UCI Machine Learning Repository. https://doi.org/10.24432/C5CG6D

The raw file is included in this repo at `data/raw/online_retail_II.xlsx`.

---

## Approach

### 1. Cleaning (`01_Demand_Forecasting.ipynb`)
Every removal is counted in `data/processed/cleaning_log.csv`.

- Removed exact duplicate rows.
- Removed codes that are not products (postage, fees, adjustments). A real product code starts with 5 digits.
- Removed rows with price ≤ 0.
- Removed cancellation invoices (invoice numbers that start with "C").
- Removed **sales that were cancelled later**. Each cancellation is matched to the original sale by the same customer, product and quantity, where the sale is on or before the cancellation date. Matching is one-to-one, in time order. Without this step, sales that never really happened would count as demand.
- Ran a second duplicate check that ignores the product description, because descriptions sometimes differ for the same row.

### 2. Weekly demand and product selection (`02_eda_weekly_demand.ipynb`)
- Summed units into **weeks starting on Monday**. Partial weeks at the start and end of the data were dropped.
- Filled weeks with no sales with **0**, including the Christmas weeks when the shop was closed. Result: a full grid of **104 weeks × 100 products**.
- Chose the **top 100 products** that were sold across the whole history. They were ranked on totals with bulk orders capped, so that one huge wholesale order cannot push a product into the top 100.
- Labelled each product's demand pattern (smooth, erratic, intermittent, lumpy) using ADI / CV².
- Looked at weekly autocorrelation (ACF) to see which lags matter.

### 3. Honest evaluation setup (`03_baselines.ipynb`)
- **Rolling-origin backtest.** Each fold trains only on the past and forecasts the next 8 weeks.
- **3 validation folds** were used for every design and tuning decision.
- **1 test fold** (10 Oct – 28 Nov 2011, the peak gift season) was **never used for any decision**. It only reports the final result: once for the Part 4 model and once for the Part 7 model, after its features had been chosen on validation.
- Main metric: **WAPE**, the total absolute error divided by total units. Also reported: MAE, Bias, MASE, and WAPE with bulk-order weeks removed.
- Baselines: naive, moving averages over 4, 8 and 13 weeks, seasonal naive, and a level-adjusted seasonal naive.

### 4. Model (`04_model.ipynb`)
- **One global LightGBM model** trained on all 100 products together, with **direct multi-horizon** forecasting: the forecast horizon `h = 1…8` is a feature.
- **Features** (`src/features.py`):
  - recent lags 1–4
  - rolling mean and standard deviation
  - the same week last year, and a smoothed version of it
  - year-over-year level ratio and seasonal ratio
  - calendar: week of year, month, UK bank holidays, weeks until Christmas
  - product ID
- Every product is **scaled by its own recent 13-week average**, so one model can learn from products that sell 20 units a week and from products that sell 2,000.
- Bulk-order spikes are **capped** before features and targets are built. The cap is computed from training data only.
- Hyperparameters were tuned with **Optuna** (30 trials) on the validation folds.
- **Three quantile models** (p10, p50, p90) give the forecast range.

### 5. Explainability and business value (`05_explain_and_business.ipynb`)
- **SHAP** values show which features drive the forecasts.
- An error analysis shows where the model is weak.
- A **stock simulation** compares "keep p90" against a classic safety-stock rule: 13-week moving average + z·σ.

### 6. SQL analysis (`06_sql_analysis.ipynb`)
- The clean data is loaded into a **SQLite** database (`data/retail.db`) with 5 tables, primary keys, foreign keys, `CHECK` rules and indexes (`sql/schema.sql`).
- 6 reusable queries in `sql/queries/` answer business questions: best sellers by units vs revenue, monthly sales with year-over-year change, revenue by country, biggest customers, and the demand patterns and seasonal profile of the 100 forecast products. Two more build model data: weekly demand (below) and weekly orders (Part 7).
- They use joins, `GROUP BY` / `HAVING`, CTEs, a recursive CTE, window functions (`RANK`, `LAG`, `SUM OVER`), `CASE WHEN` pivots and date functions.
- **Weekly demand is rebuilt with one SQL query** and matches the pandas result from Part 2 for all 10,400 product-weeks.
- The API reads its products, weekly history and weekly order counts from a small second database (`data/app.db`) with SQL.

Example (`sql/queries/top_products.sql`):
```sql
WITH product_totals AS (
    SELECT s.stock_code, p.name,
           SUM(s.quantity)          AS units,
           ROUND(SUM(s.revenue), 2) AS revenue
    FROM sales AS s
    JOIN products AS p ON p.stock_code = s.stock_code
    GROUP BY s.stock_code, p.name
)
SELECT RANK() OVER (ORDER BY units DESC)   AS units_rank,
       RANK() OVER (ORDER BY revenue DESC) AS revenue_rank,
       stock_code, name, units, revenue
FROM product_totals
ORDER BY units_rank
LIMIT :top_n;
```
The #1 product by units (*World War 2 Gliders*) is only #152 by revenue: units and money tell different stories.

### 7. Model improvement with tracked experiments (`07_experiments_mlflow.ipynb`)
- New features aimed at the two weaknesses found in Part 5:
  - **Order features**, built with SQL (`sql/queries/weekly_orders.sql`): orders per week (last 4 vs last 13 weeks), order size (last 4 vs last 13 weeks) and typical order size. Target: bulk orders and changes in how many customers buy a product.
  - **Peak index**: how strong a product's September–November season was over the last 52 weeks. Target: the under-forecast gift-season peak.
- Every experiment is tracked with **MLflow** (parameters, feature list, metrics per fold). The final models are saved in the **MLflow Model Registry** (`weekly-demand-p50`).
- Same rules as before: model settings fixed from Part 4, decisions on the 3 validation folds only, test fold used once.

| Experiment | Validation WAPE |
|---|---|
| Part 4 features (baseline) | 53.88% |
| + peak index | 54.11% |
| **+ order features** | **53.54%** |
| + peak index + order features | 53.65% |

- The gain from order features held in **13 of 15** checks with different random seeds, so it is small but not luck.
- **Test fold (used once): WAPE 53.4% → 52.9%**, MAE 160.2 → 158.5, p10–p90 coverage 81.5%.
- **What did not work:** the peak index could not help on validation, because no validation fold contains a gift season. The Christmas under-forecast is **not** fixed (bias about −18.6%). With only one earlier gift season in the data, the model cannot learn how big a peak can get. Bulk orders are not fixed either: Part 8 shows the gain comes from normal weeks.
- All runs are also saved to `data/processed/feature_experiments.csv`.
- **Deployed:** the model with order features is now the one the live app uses (`models/forecast_bundle.pkl`). Its weekly order counts are stored in the `weekly_orders` table of `data/app.db`, so the chain **SQL → features → model → API** runs end to end. The Part 4 model is kept as `models/forecast_bundle_v1.pkl`, and notebooks 04–05 still analyse that model.

### 8. Statistical tests (`08_statistical_tests.ipynb`)
A single test score depends on which products and which weeks happened to be in the test. Two checks show how much the results can be trusted (functions in `src/stats.py`):
- **Bootstrap confidence intervals:** the 100 test products are resampled with replacement 10,000 times (each product keeps its 8 weeks). A **paired** version compares two models on the same resampled products.
- **Diebold-Mariano test:** compares the weekly errors of two models over the 32 forecast weeks of all four folds, allowing for errors up to 7 weeks apart to be correlated, with the Harvey–Leybourne–Newbold small-sample correction.

| Comparison | WAPE difference (test fold) | Paired bootstrap 95% CI (test fold) | Diebold-Mariano p-value (32 weeks) |
|---|---|---|---|
| LightGBM (Part 7) vs moving average | −5.1 pp | −7.7 to −3.0 pp | **0.002** |
| LightGBM (Part 4) vs moving average | −4.5 pp | −7.1 to −2.4 pp | **0.013** |
| Part 7 vs Part 4 (order features) | −0.5 pp | −1.0 to −0.1 pp | 0.07 |

- **LightGBM beats the moving average in both tests.** This is the main claim of the project, and it holds.
- **The order-feature gain is small:** it is consistent across products (better in 99% of bootstrap samples) but **not significant over time** (p ≈ 0.07), so it is described as a small improvement, not a big one.
- **The gain is not in bulk-order weeks:** about 1,200 of the 1,300 units of lower test error come from normal weeks. The 31 bulk-order product-weeks still cause about 28% of the error, and every model forecasts them 77–80% too low.
- The deployed model's test WAPE of 52.9% has a 95% CI of **48.0%–57.7%**, and its p10–p90 coverage interval (77.6%–85.1%) contains the 80% target.

---

## Results (held-out test, 10 Oct – 28 Nov 2011)

| Metric | Moving average (13 wk) | LightGBM (Part 4) | **LightGBM + order features (Part 7, deployed)** |
|---|---|---|---|
| WAPE (lower is better) | 57.9% | 53.4% | **52.9%** |
| WAPE 95% CI (bootstrap, Part 8) | 52.2–64.0% | 48.6–58.3% | 48.0–57.7% |
| p10–p90 coverage (target 80%) | — | 81.8% | 81.5% |
| Bias | — | −18.3% | −18.6% |

The live app uses the Part 7 model. The business result below comes from the Part 5 stock simulation, which uses the Part 4 model.

**Business result:** to reach a **90% service level**, keeping p90 as stock needed **~11% less stock** and caused **~35% fewer lost sales** than the classic safety-stock rule.

### Limitations (being honest)
- **Peak weeks are under-forecast.** Bias is −18% on the test period. The test period is the run-up to Christmas, and the model only has one previous Christmas to learn from. This is why the app recommends stocking to **p90**, not to the median.
- **Bulk orders are under-forecast.** One-off wholesale orders are about 4% of product-weeks but about 28% of the error. They are capped in training, and the Part 7 order features did not fix them (Part 8). Sales history cannot show such an order before it happens; advance orders from wholesale customers would be needed.
- **Only 2 years of history.** That means just one "same week last year" for each week.
- **The simulation is simplified.** It has no supplier lead times, and unsold stock does not carry over to the next week.
- **No cold start.** The model only covers the 100 products it was trained on.
- WAPE is still high in absolute terms, because weekly demand for single gift products is very noisy.

### Future work
- More years of data, so a validation fold can sit inside a gift season and peak features (like the Part 7 peak index) can be tested properly
- A seasonal multiplier for gift-season weeks, since tree models cannot forecast above what they have seen
- Croston / TSB models for intermittent products
- A stock simulation with lead times and stock carried over between weeks
- Forecasts for new products

---

## Tests

```bash
pip install -r requirements-dev.txt
python -m pytest -v      # 30 passed
```

**Continuous integration** (`.github/workflows/ci.yml`): GitHub Actions runs on every push and pull request to `main`:
1. **tests**: installs the pinned packages and runs the full pytest suite on Linux.
2. **docker**: builds the Docker image, starts the container, and calls `/health`, `/products` and `/forecast` (including the 404 case). It only runs if the tests pass.

- **No leakage:** changing future weeks (units or orders) does not change any feature.
- **Train/serve consistency:** the API builds exactly the same features as training did, including the order features.
- **Model bundle and output checks:** the app's bundle has the order features and order counts it needs, all forecasts are ≥ 0 and p10 ≤ p50 ≤ p90, and the Part 4 bundle still works for notebooks 04–05.
- **API checks:** `/`, `/health`, `/products` and `/forecast` work, an unknown product returns 404, and an invalid number of weeks returns 422.
- **Database checks:** the history and order counts in `data/app.db` are identical to the training data and give identical forecasts, the database is opened read-only, the weekly demand SQL query follows the Part 2 rules on a small hand-made example, and the schema rejects bad rows.
- **Statistics checks:** with no lags the Diebold-Mariano test equals a paired t-test, it detects a clearly better forecast, and the bootstrap is reproducible, centred on the test score and uses the same products for every model.

---

## API

| Method | Endpoint | What it does |
|---|---|---|
| GET | `/` | Stock Planner web page |
| GET | `/health` | Status, training date, number of products, test metrics |
| GET | `/products` | List of the 100 products (code + name) |
| POST | `/forecast` | Forecast for one product, plus its last 26 weeks of history |

Example:
```bash
curl -X POST https://demand-forecasting-o8yb.onrender.com/forecast \
  -H "Content-Type: application/json" \
  -d '{"stock_code": "84077", "weeks": 3}'
```

---

## Project structure

```
Demand_Forecasting/
├── 01_Demand_Forecasting.ipynb     # load, audit, clean
├── 02_eda_weekly_demand.ipynb      # weekly demand, product selection, EDA
├── 03_baselines.ipynb              # backtest folds, metrics, baselines
├── 04_model.ipynb                  # LightGBM, Optuna, quantiles, test, bundle
├── 05_explain_and_business.ipynb   # SHAP, error analysis, stock simulation
├── 06_sql_analysis.ipynb           # SQL database, business queries, weekly demand in SQL
├── 07_experiments_mlflow.ipynb     # new features, experiments tracked with MLflow
├── 08_statistical_tests.ipynb      # bootstrap confidence intervals, Diebold-Mariano test
├── src/
│   ├── features.py                 # feature engineering (shared by training and API)
│   ├── evaluation.py               # metrics and baseline helpers
│   ├── stats.py                    # Diebold-Mariano test and bootstrap helpers
│   ├── forecast.py                 # makes forecasts from the saved bundle
│   └── db.py                       # SQLite helpers (connect, run saved queries, load app data)
├── sql/
│   ├── schema.sql                  # tables of data/retail.db
│   ├── app_schema.sql              # tables of data/app.db
│   └── queries/                    # one .sql file per business question
├── scripts/build_db.py             # builds both SQLite databases
├── app/
│   ├── main.py                     # FastAPI app
│   ├── schemas.py                  # request / response models
│   └── static/index.html           # Stock Planner UI
├── models/forecast_bundle.pkl      # Part 7 model + everything the API needs (used by the app)
├── models/forecast_bundle_v1.pkl   # Part 4 model (used by notebooks 04–05)
├── data/raw/                       # original Excel file
├── data/processed/                 # cleaned and weekly data
├── data/app.db                     # small SQLite database read by the API
├── data/retail.db                  # full SQLite database (not in git, built by scripts/build_db.py)
├── tests/                          # pytest tests
├── .github/workflows/ci.yml        # CI: tests + Docker build and API smoke test
├── Dockerfile                      # container image for the API
├── .dockerignore
├── requirements.txt
├── requirements-dev.txt            # requirements.txt + test packages
└── .python-version                 # 3.13.9
```

---

## Run it locally (Windows / PowerShell)

```powershell
git clone https://github.com/Akashbaloni/Demand-Forecasting.git
cd Demand-Forecasting
python -m venv venv
.\venv\Scripts\Activate.ps1
pip install -r requirements.txt
uvicorn app.main:app --reload
```

Then open http://127.0.0.1:8000.

### With Docker

```powershell
docker build -t demand-forecast .
docker run -p 8000:8000 demand-forecast
```

Then open http://localhost:8000. The image contains only what the API needs (`app/`, `src/`, `models/`, `data/app.db`), runs as a non-root user, and has a health check on `/health`.

To re-run the notebooks, also install the notebook-only packages (jupyter, matplotlib, optuna, shap, openpyxl, plus anything else imported at the top of each notebook) and run them in order, 01 → 05.

Notebooks 06 and 07 need the full SQL database, which is too big for GitHub. Build it first (takes about 10 seconds):

```powershell
python scripts/build_db.py
```

Notebook 07 also needs `mlflow`. It saves the app's model to `models/forecast_bundle.pkl`; after re-running it, run `python scripts/build_db.py` again so `data/app.db` matches the new model. Its runs are stored in a local `mlflow.db` file; to browse them:

```powershell
mlflow ui --backend-store-uri sqlite:///mlflow.db
```

Notebook 08 only needs the two saved models in `models/`, not the SQL database.

---

## Tech stack
Python 3.13 · pandas · SQL (SQLite) · LightGBM · Optuna · SHAP · MLflow · SciPy (statistical tests) · FastAPI · Pydantic · pytest · Docker · GitHub Actions (CI) · Render