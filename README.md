# Multi-Country Influenza Forecasting & Epidemic Spike Detection

A multi-country surveillance, preprocessing, and feature engineering pipeline built on WHO FluNet and FluID weekly surveillance data. This project processes historical data across four distinct seasonality regimes (**USA**, **UK**, **India**, and **Japan**) to prepare data for 5-year probabilistic outbreak/spike forecasting models.

---

## Project Overview

The primary objective of this project is **outbreak/spike detection**—predicting the probability that a given week's case count will exceed a country-specific epidemic threshold—rather than point forecasting alone. 

### Seasonality Coverage
- **USA & UK**: Temperate Northern Hemisphere regimes with defined winter spikes.
- **India**: Tropical/subtropical regime with non-sharply defined, multi-peak seasonality.
- **Japan**: Temperate Asian regime with strong reporting infrastructure.

---

## Preprocessing Pipeline Architecture

The raw source data consists of multi-tab weekly virological surveillance records (`India`, `UK`, `Japan`, `USA`). The preprocessing pipeline handles real-world epidemiological data quality issues:

1. **Load and Tag**: Concatenates individual country sheets into a unified long-format dataset tagged with country identifiers.
2. **Temporal Alignment**: Discards raw, misaligned `ISO_WEEK` values. Parses `ISO_SDATE` strings explicitly into ISO-8601 datetimes and derives self-consistent `year` and `iso_week` attributes.
3. **Numeric Harmonization**: Explicitly casts all numeric case-count and subtype columns to uniform numeric types, coercing invalid entries to `NaN`.
4. **Surveillance Stream Consolidation**: Groups records by `(country, year, iso_week)` and sums case counts across duplicate reporting streams (`SENTINEL`, `NONSENTINEL`, `NOTDEFINED`).
5. **Missing Value Strategy**:
   - **Subtype Counts**: Blank subtype counts (`AH1`, `AH3`, `BVIC`, etc.) post-aggregation are zero-filled (indicating absence of strain detection).
   - **Case Totals**: Reconstructs missing `INF_ALL` totals from `INF_A + INF_B` where available; preserves true missing reporting periods as explicit `NaN` entries.
6. **Calendar Gap Exposure**: Reconstructs a full weekly time grid for each country's temporal range to ensure unreported weeks are explicitly represented rather than silently dropped.
7. **COVID-19 Regime Flagging**: Marks years 2020–2022 (`covid_period = True`) to isolate pandemic-disrupted seasonality from baseline models.
8. **Epidemic Threshold & Outbreak Labeling**: Computes a per-country historical 90th percentile baseline (`epidemic_threshold`) across non-COVID seasons for each ISO week, generating a binary classification target (`is_spike`).

---

## Dataset Schema (`cleaned_weekly_flu_panel_fully_clean.csv`)

| Column Name | Type | Description |
| :--- | :--- | :--- |
| `country` | string | Country identifier (`India`, `Japan`, `UK`, `USA`) |
| `iso_sdate_parsed` | datetime | Parsed start date of the surveillance week (ISO-8601) |
| `year` | integer | Derived ISO calendar year |
| `iso_week` | integer | Derived ISO calendar week (1–52/53) |
| `AH1`, `AH1N12009`, `AH3`, `AH5`, `ANOTSUBTYPED` | float | Subtype case counts for Influenza A strains |
| `BVIC`, `BYAM`, `BNOTDETERMINED` | float | Lineage case counts for Influenza B strains |
| `INF_A`, `INF_B` | float | Aggregate positive counts for Influenza A and B |
| `INF_ALL` | float | Total positive influenza cases (Primary target) |
| `INF_NEGATIVE` | float | Total negative specimens processed |
| `SPEC_RECEIVED_NB`, `SPEC_PROCESSED_NB` | float | Total specimens received and processed |
| `covid_period` | boolean | Flag for COVID disruption period (`2020–2022`) |
| `epidemic_threshold` | float | Historical 90th percentile case threshold for country-week |
| `is_spike` | boolean | Outbreak target label (`INF_ALL > epidemic_threshold`) |

---

## Pipeline Execution

To run the preprocessing script and generate the modeling-ready panel dataset:

```bash
# Clone the repository
git clone [https://github.com/sambitkarmakar03/Flu-forecasting.git](https://github.com/sambitkarmakar03/Flu-forecasting.git)
cd Flu-forecasting

# Run preprocessing notebook or script
python Preprocess/preprocess.py  # or execute Preprocess/preprocess.ipynb


# Exploratory Data Analysis (EDA) — Time Series Influenza Forecasting

This directory contains the exploratory data analysis pipeline for the Time Series Influenza Forecasting project. The objective of this phase is to clean, transform, and analyze global and regional influenza surveillance data to uncover temporal trends, seasonality patterns, and data quality issues prior to predictive modeling.

---

## 📁 Directory Overview

```text
Preprocess/
├── data_analysis.ipynb      # Primary Jupyter Notebook for data cleaning, visualization, and EDA


pip install pandas numpy matplotlib seaborn plotly

