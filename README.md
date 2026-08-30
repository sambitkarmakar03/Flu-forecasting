# Multi-Country Influenza Forecasting & Epidemic Spike Detection

A comprehensive, multi-country surveillance, feature engineering, modeling, and evaluation pipeline built on WHO FluNet and FluID weekly surveillance data. This project processes historical surveillance across four distinct epidemiological seasonality regimes (**USA**, **UK**, **India**, and **Japan**) to engineer, train, and benchmark probabilistic outbreak detection and 52-week time-series forecasting models.

---

## 📌 Project Overview

The primary objective of this project is **outbreak and epidemic spike detection**—predicting the probability that a given week's case count ($y_{c,t}$) will exceed a country-specific baseline threshold ($\tau_{c,w}$)—alongside multi-step continuous target forecasting (`INF_ALL`).

### Target Definition & Thresholding

* **Primary Forecast Target (`INF_ALL`)**: Continuous log-transformed positive influenza cases ($y_{\text{log}} = \log(1+y)$).
* **Binary Outbreak Target (`is_spike`)**: $\mathbb{I}(y_{c,t} > \tau_{c,w})$, where $\tau_{c,w}$ is the historical 90th percentile case threshold calculated per country ($c$) and ISO week ($w$) over pre-COVID historical seasons (excluding 2020–2022).

### Seasonality Coverage & Epidemiological Regimes

* **USA & UK**: Temperate Northern Hemisphere regimes characterized by sharply defined winter spikes (Weeks 50–10).
* **India**: Tropical/subtropical regime featuring complex multi-peak (bimodal) seasonality driven by monsoon and winter dynamics.
* **Japan**: Temperate Asian regime with strong reporting infrastructure and distinct seasonal waves.

---

## 🏗️ Repository Architecture

```text
Flu-forecasting/
├── Data/
│   └── cleaned_weekly_flu_panel_fully_clean.csv
├── Preprocess/
│   ├── preprocess.py              # Raw data cleaning, missing value exposure & aggregation
│   ├── preprocess.ipynb           # Interactive preprocessing pipeline
│   └── data_analysis.ipynb        # EDA, visual analytics, and seasonality profiling
├── Models/
│   ├── baseline_sarima.py         # Country-specific univariate SARIMA benchmark
│   └── global_lgbm.py             # Joint pooled LightGBM multi-step regressor
├── README.md                      # Complete project documentation
└── requirements.txt               # Dependencies

```

---

## 🧹 Preprocessing Pipeline Architecture

The raw source data consists of multi-tab weekly virological surveillance records (`India`, `UK`, `Japan`, `USA`). The automated cleaning pipeline addresses missing values, reporting inconsistencies, and structural gaps:

1. **Load and Tag**: Ingests individual sheet data and concatenates records into a unified long-format dataset tagged with `country`.
2. **Temporal Alignment**: Standardizes dates to ISO-8601 datetimes (`iso_sdate_parsed`) and extracts uniform `year` and `iso_week` keys.
3. **Numeric Harmonization**: Coerces case counts and subtype data into explicit float types, handling non-numeric reporting artifacts.
4. **Stream Consolidation**: Aggregates duplicate surveillance streams (`SENTINEL`, `NONSENTINEL`, `NOTDEFINED`) per `(country, year, iso_week)`.
5. **Missing Value Reconstruction**:
* Zero-fills subtype counts (`AH1`, `AH3`, `BVIC`, `BYAM`) post-aggregation when absent from positive detections.
* Reconstructs missing aggregate counts (`INF_ALL`) via component sum ($INF\_A + INF\_B$) where applicable, preserving true structural missingness.


6. **Full Time Grid Exposure**: Reconstructs missing calendar weeks to ensure complete continuous time-series grids across all regions.
7. **COVID-19 Disruption Isolation**: Flags years 2020–2022 (`covid_period = True`) to prevent severe pandemic reporting anomalies from distorting baseline thresholds.
8. **Thresholding & Binary Target Generation**: Evaluates country-week specific 90th percentiles ($\tau_{c,w}$) to define the `is_spike` label.

---

## 🏷️ Dataset Schema (`cleaned_weekly_flu_panel_fully_clean.csv`)

| Column Name | Type | Description |
| --- | --- | --- |
| `country` | string | Country identifier (`India`, `Japan`, `UK`, `USA`) |
| `iso_sdate_parsed` | datetime | Parsed start date of the surveillance week (ISO-8601) |
| `year` | integer | Derived ISO calendar year |
| `iso_week` | integer | Derived ISO calendar week (1–52/53) |
| `AH1`, `AH1N12009`, `AH3`, `AH5` | float | Subtype case counts for Influenza A strains |
| `BVIC`, `BYAM`, `BNOTDETERMINED` | float | Lineage case counts for Influenza B strains |
| `INF_A`, `INF_B` | float | Aggregate positive counts for Influenza A and B |
| `INF_ALL` | float | Total positive influenza cases (**Primary Target**) |
| `INF_NEGATIVE` | float | Total negative specimens processed |
| `SPEC_RECEIVED_NB`, `SPEC_PROCESSED_NB` | float | Total specimens received and processed |
| `covid_period` | boolean | Disruption flag for pandemic period (`2020–2022`) |
| `epidemic_threshold` | float | Baseline 90th percentile threshold ($\tau_{c,w}$) |
| `is_spike` | boolean | Outbreak classification label (`INF_ALL > epidemic_threshold`) |

---

## ⚙️ Modeling Methodology

### 1. Feature Engineering

* **Log Transformation**: Target values are transformed via $y_{\text{log}} = \log(1+y)$ (`np.log1p`) to stabilize high-variance outbreak spikes. Forecasts are inverted using $\hat{y} = \exp(\hat{y}_{\text{log}})-1$ (`np.expm1`).
* **Autoregressive Lags & Rolling Statistics**: Extract $\text{lag}_1, \text{lag}_2, \text{lag}_3$, and $\text{lag}_{52}$, alongside a 4-week moving average (`rolling_mean_4`) and standard deviation (`rolling_std_4`).
* **Harmonic Seasonal Encodings**: Captures annual (52-week) and semi-annual (26-week) multi-peak cycles:
* $\text{week\_sin\_52} = \sin\left(\frac{2\pi \cdot \text{week}}{52}\right)$, $\text{week\_cos\_52} = \cos\left(\frac{2\pi \cdot \text{week}}{52}\right)$
* $\text{week\_sin\_26} = \sin\left(\frac{2\pi \cdot \text{week}}{26}\right)$, $\text{week\_cos\_26} = \cos\left(\frac{2\pi \cdot \text{week}}{26}\right)$


* **Country Interaction Features**: Multiplies seasonal harmonic encodings with target country flags (e.g., `week_sin_52_x_country`) to enable pooled global models to capture regional seasonal timing.

### 2. Backtesting & Forecast Horizon

* **Rolling-Origin Evaluation**: An $N$-fold time-series backtest ($N = 6\text{--}7$ folds) with a forecast horizon of $H = 12$ weeks per fold.
* **Panel Cutoffs**: Datasets are partitioned using globally aligned temporal cutoff dates to strictly prevent temporal data leakage.
* **Recursive Forecasting**: Multi-step predictions up to 52 weeks ahead append predictions iteratively back into history to dynamically re-evaluate lag and rolling features.

$$\boxed{ \hat{y}_{t} \rightarrow \text{Update History} \rightarrow \text{Recalculate Features} \rightarrow \hat{y}_{t+1} \rightarrow \cdots }$$

### 3. Evaluated Models

* **Baseline SARIMA**: Independent per-country univariate $\text{SARIMA}(1,1,1) \times (1,1,1)_{52}$ fitted on log-transformed historical targets.
* **Global LightGBM Regressor**: A single decision tree ensemble (`LGBMRegressor`) trained across pooled multi-country data:
* **Objective**: Regression (`RMSE`)
* **Parameters**: `n_estimators=300`, `learning_rate=0.03`, `num_leaves=31`, `random_state=42`



---

## 📊 Forecasting Model Benchmark & Performance Analysis

### 1. Overall Benchmark Summary

Across a multi-fold rolling-origin backtest ($N = 6\text{--}7$ folds per country, $H = 12$ weeks), the **joint LightGBM regression model** consistently outperformed individual country-specific **SARIMA baselines**.

* **Statistical Significance**: LightGBM achieved a statistically significant improvement across **100% of tested territories** ($p < 0.05$ and $\text{DM stat} > 0$).
* **Relative Scale Error (MASE)**: Reduced average relative error by **36.9% to 71.7%** across all panel countries.
* **Total Volume Error (WAPE)**: Substantially reduced total absolute volume misallocation across high-magnitude series.

### 2. Country-by-Country Performance Analytics

| Country | Backtest Horizon | Baseline (SARIMA MASE) | Model (LightGBM MASE) | MASE Error Reduction | Baseline (SARIMA WAPE) | Model (LightGBM WAPE) | WAPE Improvement | DM Stat (z) | DM p-value | Statistically Significant? |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| **India** | 7 Folds (84 Wks) | `0.425` | `0.239` | **↓ 43.8%** | `105.3M` | `90.6M` | **↓ 14.0%** | `+2.485` | `0.013` | **Yes ($p < 0.05$)** |
| **Japan** | 6 Folds (72 Wks) | `0.146` | `0.055` | **↓ 62.3%** | `131.8M` | `97.8M` | **↓ 25.8%** | `+1.975` | `0.048` | **Yes ($p < 0.05$)** |
| **UK** | 6 Folds (72 Wks) | `0.065` | `0.041` | **↓ 36.9%** | `118.5M` | `22.0M` | **↓ 81.4%** | `+2.240` | `0.025` | **Yes ($p < 0.05$)** |
| **USA** | 6 Folds (72 Wks) | `0.059` | `0.017` | **↓ 71.7%** | `56.3%` | `24.1%` | **↓ 57.2%** | `+2.197` | `0.028` | **Yes ($p < 0.05$)** |

---

### 3. Analytics Breakdown & Strategic Insights

* **India (Complex Bimodal Epidemics)**: MASE decreased from `0.425` to `0.239` (**↓ 43.8%**). While standard SARIMA struggled with two distinct annual peaks (monsoon and winter), LightGBM effectively captured both using **26-week harmonic interaction terms**.
* **Japan & UK (Off-Peak Stability and Peak Control)**: Japan achieved a **62.3% MASE reduction**, while the UK saw an **81.4% WAPE reduction** (`118.5M` to `22.0M`). LightGBM eliminated off-peak baseline drift common in SARIMA transition phases.
* **USA (High-Volume Precision)**: MASE improved by **71.7%** (`0.059` to `0.017`) and WAPE by **57.2%** (`56.3%` to `24.1%`), proving the pooled tree-based approach handles non-linearities and high-magnitude variance better than linear autoregressive baselines.

---

## ⚡ Setup & Execution

### 1. Installation

```bash
# Clone repository
git clone https://github.com/sambitkarmakar03/Flu-forecasting.git
cd Flu-forecasting

# Create and activate virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt

```

### 2. Running Data Pipeline, EDA, & Models

```bash
# Execute raw data preprocessing pipeline
python Preprocess/preprocess.py

# Launch EDA analysis
jupyter notebook Preprocess/data_analysis.ipynb

# Train baseline and joint global models
python Models/baseline_sarima.py
python Models/global_lgbm.py

```
