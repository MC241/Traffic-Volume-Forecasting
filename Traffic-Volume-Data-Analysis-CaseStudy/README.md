# Hourly Traffic Volume Forecasting (I-94, Minneapolis–St Paul)

Forecasting hourly highway traffic from calendar, holiday and weather data.
Originally an internship case study at BI4ALL (2023); rebuilt in 2026 with a data-quality-first approach.

## Key results (test set: Jan–Sep 2018, 6,533 observed hours)

| Model | Test WAPE |
|---|---|
| 2023 pipeline (reproduced on the same data) | 13.2% |
| Baseline: same hour last week | 10.2% |
| Baseline: median by weekday × hour | 7.6% |
| **XGBoost, calendar features only (final)** | **6.7%** |
| XGBoost, calendar + weather + lags | 6.8% |

WAPE = total absolute error ÷ total actual traffic. It is used instead of MAPE because night-time volumes near zero make MAPE explode.

## What I found

**1. Data quality mattered more than the algorithm.** Profiling showed 23% of hours missing, including a ten-month outage (Aug 2014 – Jun 2015). The original pipeline back-filled the target, so 26% of training rows were invented. Holidays were labelled on the midnight hour only, so the holiday flag covered 1 hour per holiday instead of 24.

**2. Traffic is a calendar problem.** Once hour, weekday and holiday are modelled properly, a model with no weather and no recent traffic is the most accurate. It can also forecast any future date, not only the next 7 days.

**3. Weather adds nothing on average, and the weather feed is unreliable.** The worst-predicted days were 14–15 April 2018, a blizzard that halved traffic. The weather source labels those hours as *Snow* but records 0.0 mm of snowfall. Only 63 of 48,204 rows have any snow amount at all.

## What changed from 2023

| 2023 | 2026 |
|---|---|
| `bfill()` on every column, including the target | Missing hours stay missing; never impute the target |
| Holiday flag on midnight row only | Holiday applied to the whole day |
| No hour / weekday features | Explicit calendar features |
| 30-day lag (misaligns weekdays) | 1, 2 and 4-week lags (same weekday) |
| No baseline | Two baselines every model must beat |
| One hyperparameter combination per "grid" | Real grid search, `TimeSeriesSplit` with a 1-week gap |
| Metrics include imputed hours | Metrics on observed hours only |

## Repository

```
traffic_volume_forecasting.ipynb   full analysis, executed with outputs
data/                              UCI dataset (gzipped CSV)
results/model_comparison.csv       results table produced by the notebook
```

## How to run

```bash
pip install -r requirements.txt
jupyter notebook traffic_volume_forecasting.ipynb
```

Tested with Python 3.12, pandas 3.0, scikit-learn 1.9, xgboost 3.4. Results are reproducible with `random_state=2`.

## Limitations

- Weather inputs are observed, not forecast. A deployed model would only have weather forecasts.
- No event or road-works calendar, which likely explains several of the largest errors.
- One road segment, one direction: results may not transfer to other locations.

## Data

[Metro Interstate Traffic Volume](https://archive.ics.uci.edu/dataset/492/metro+interstate+traffic+volume), UCI Machine Learning Repository, licensed CC BY 4.0. Traffic data from MN DoT, weather from OpenWeatherMap.
