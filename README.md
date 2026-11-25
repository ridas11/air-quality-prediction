# UK Air Quality Prediction: NO2 at London Marylebone Road

Forecasting hourly nitrogen dioxide (NO2) at a busy London roadside monitoring site, using DEFRA data, time and lag features, and weather. The project compares several models against honest baselines at two horizons (1 hour and 24 hours ahead) and reports where the models help and where they don't.

**Tech stack:** Python, pandas, NumPy, scikit-learn, XGBoost, matplotlib, Open-Meteo API, Git/GitHub

## Headline results

| Task | Best baseline (MAE) | Best model (MAE) | Improvement |
|---|---|---|---|
| 1 hour ahead | Persistence: 5.72 | XGBoost: 5.27 | ~8% |
| 24 hours ahead, no weather | Hour-of-day average: 11.68 | Linear Regression: 10.43 | ~11% |
| 24 hours ahead, with weather | Hour-of-day average: 11.68 | Linear Regression: 9.03 | ~23% |

MAE is in µg/m³. Mean NO2 over the period is about 25 µg/m³.

- At a 1-hour horizon almost all the predictive power comes from "NO2 an hour ago"; the fancier models add little over linear regression.
- At 24 hours persistence fails (MAE 12.5, no better than predicting the mean), so the useful baseline is the average NO2 for that hour of day.
- Adding weather (wind, temperature, precipitation) improved all three models by 13-14% at the 24-hour horizon.
- The three models are effectively tied once simplified, so I don't claim a winner at 24 hours.

## Data

- **Source:** [DEFRA UK-AIR](https://uk-air.defra.gov.uk/data/) data selector, AURN network, London Marylebone Road (urban traffic site)
- **Variables:** hourly NO2, PM10, PM2.5; 9 Aug 2025 to 10 Aug 2026 (8,784 hourly rows)
- **Weather:** hourly temperature, wind speed, wind direction and precipitation from the [Open-Meteo](https://open-meteo.com/) historical archive
- About 64% of readings were ratified ("V") and 36% not yet verified ("N") at download time. No readings were flagged as suspect. Both were kept, since "not verified" means final quality checks are pending rather than a known fault.

## Cleaning

- Skipped the metadata rows at the top of the DEFRA CSV and dropped the trailing "End" row
- Converted pollutant columns to numeric; non-numeric entries became missing values
- DEFRA labels midnight as `24:00:00`; these 366 rows were converted to `00:00:00` on the following day
- Interpolated gaps of up to 3 hours only. Longer gaps were left missing rather than invented
- PM2.5 has a 20-day outage (19 Feb to 10 Mar 2026) plus several shorter ones, which is why NO2 was chosen as the prediction target

## Exploratory findings

- **Daily cycle:** NO2 bottoms out around 3-4am (about 15 µg/m³) and peaks around 5-6pm (about 32 µg/m³), consistent with rush-hour traffic
- **Weekly cycle:** weekdays average about 24.5-28 µg/m³, Saturday about 23.5 and Sunday about 21
- **Seasonality:** PM2.5 is higher in winter and lower in summer
- **Correlations:** PM10 and PM2.5 correlate at 0.59, NO2 and PM10 at 0.43, NO2 and PM2.5 at 0.25
- **Anomaly check:** May 2026 had a high NO2 monthly mean despite near-complete coverage, driven by two isolated spikes (about 120 and 137 µg/m³) rather than a sustained shift

## Modelling approach

- **Features:** hour and month (sine/cosine encoded), weekend and rush-hour flags, 1-hour and 24-hour lags, 6-hour and 24-hour rolling means. Rolling features are shifted by one hour so the current value never leaks in
- **Split:** chronological 80/20, never random. Test period is late May to early August 2026
- **Models:** Linear Regression, Random Forest, XGBoost
- **Baselines:** training mean, persistence, and average-for-that-hour-of-day
- **Metrics:** MAE (same units as the pollutant) and RMSE (penalises large misses)

### 1 hour ahead

| Model | MAE | RMSE |
|---|---|---|
| Training mean (baseline) | 12.38 | 15.95 |
| Persistence (baseline) | 5.72 | 9.39 |
| Linear Regression | 5.43 | 8.69 |
| Random Forest | 5.32 | 8.35 |
| XGBoost | 5.27 | 8.26 |

Errors are lowest overnight and highest from 11am to 5pm, peaking around 3pm.

### 24 hours ahead

| Model | MAE (no weather) | MAE (with weather) |
|---|---|---|
| Training mean (baseline) | 12.75 | n/a |
| Persistence (baseline) | 12.52 | n/a |
| Hour-of-day average (baseline) | 11.68 | n/a |
| Linear Regression | 10.43 | 9.03 |
| Random Forest (shallow) | 10.61 | 9.07 |
| XGBoost (shallow) | 10.68 | 9.30 |

An initial run with the month features and deeper trees gave Linear Regression 10.50, Random Forest 11.93 and XGBoost 13.84, worse than the hour-of-day baseline. Removing the month features and using shallower trees brought XGBoost to 10.68. Mean NO2 was similar in train and test (25.7 vs 25.4), so this was not a simple level shift. I changed both things at once, so I can't say which fix did the work.

### Which weather variable matters?

> TODO: paste the feature-importance chart and the leave-one-out table here once run.

## Limitations

- **One site, one year.** Results may not transfer to background or rural sites, and the model has seen each season only once
- **One test period.** A single summer-heavy chronological split. Differences of a few tenths of MAE between models are within noise. Time-series cross-validation would be a stronger check
- **Weather is the actual weather at the target hour**, which you would only have with a perfect forecast. A deployed system would use forecast weather and score worse, so the weather results are an upper bound
- **Timestamp alignment.** DEFRA times are hour-ending and Open-Meteo times are on the hour, a small mismatch I did not correct
- **Unverified data.** About 36% of readings were not yet ratified
- **Interpolation** of short gaps slightly smooths the series, and rows with missing targets were dropped

## Next steps

- Time-series cross-validation across several folds
- Use forecast rather than observed weather
- Add traffic or holiday data
- Extend to other AURN sites

## How to run

1. Download the data from the DEFRA data selector (AURN, London Marylebone Road, NO2, PM10, PM2.5, measured, hourly) and place the CSV in `data/raw/`
2. `pip install -r requirements.txt`
3. Open `notebooks/` and run the notebook top to bottom

## Repository layout

```
air-quality-prediction/
├── README.md
├── requirements.txt
├── data/raw/
└── notebooks/
```
