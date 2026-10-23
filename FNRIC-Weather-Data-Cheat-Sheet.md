# FNRIC Weather Data: Hackathon Cheat Sheet

Oct 22, 2026 · Carson Black

Everything you need to pull California chill and growing degree day (GDD) data from the Fruit & Nut Research & Information Center (FNRIC), check your math against it, and extend it with a forecast.

## Chill and GDD in five minutes

Two counters run at different times of year: chill builds up during winter dormancy, and GDD builds up after bloom.

**Chill.** Fruit and nut trees need enough winter cold before they'll leaf out and bloom normally. Too little chill means late, uneven bloom and poor fruit set. FNRIC's chill season runs Sept 1–Aug 31, labeled like `2025-2026`. Every model works on **hourly air temperature (°F)**:

| Model | One hour counts as | API field |
| --- | --- | --- |
| Hours below 45°F | 1 if temp ≤ 45, else 0 | `hours_lt_45` |
| Hours 32–45°F | 1 if 32 < temp ≤ 45, else 0 | `hours_32_45` |
| Utah | ≤ 34: 0 · 34–36: 0.5 · 36–48: 1 · 48–54: 0.5 · 54–60: 0 · 60–65: −0.5 · > 65: −1 (warm hours subtract) | `units_utah` |
| Dynamic (chill portions) | Two-step model: cold builds an intermediate that warmth can destroy until it locks in as a portion. Preferred for California's mild winters. | `portions_dynamic` |

**GDD (growing degree days).** After bloom, development speed tracks warmth. Each day contributes the heat between a lower and an upper threshold. FNRIC uses the **single-sine method** (Zalom et al. 1983), estimating the day's temperature curve from its min and max, with **lower 5°C (41°F) and upper 35°C (95°F)**, in °C·days. Summed from bloom, GDD predicts events like almond hull split.

A simpler approximation, fine for a first pass:

```math
\mathrm{GDD}_{day} = \max\left(0,\ \frac{\min(T_{max}, 35) + \max(T_{min}, 5)}{2} - 5\right)
```

Some models count **growing degree hours (GDH)** instead, summing hourly heat. FNRIC uses these for pistachio and peach models.

## The FNRIC API

It's public, needs no key, sends CORS headers for any origin (so browser apps on localhost work), and returns JSON unless noted. Base URL (PLEASE DO NOT ABUSE/SPAM! We pay money from limited budgets for the resources.):

```
https://f5dby14cj5.execute-api.us-west-2.amazonaws.com/prod/
```

Station IDs look like `CIMIS-6` (Davis) or `Wildeye-111090` (in-orchard CPRB stations). Get the list from `stations`.

| Endpoint | Params | Returns |
| --- | --- | --- |
| `GET stations` | none | `{stations:[…]}`: 303 stations (277 CIMIS, 26 Wildeye; 172 active). Fields: `StationID`, `Name`, `County`, `ConnectDate`, `IsActive`, `HmsLatitude`, `HmsLongitude` |
| `GET chill-models-download` | `station-id` | **CSV** of the station's full daily chill history (Davis goes back to 1982): `Date`, `hours_lt_45`, `hours_32_45`, `units_utah`, `portions_dynamic`. **Start here.** |
| `GET chill-models` | `station-id`, `chill-year` (`2025-2026`) | `{model_values:[…]}`: one row per day with `Date`, `hours_lt_45`, `hours_32_45`, `units_utah`, `portions_dynamic`, `n_hours` (hours of data that day) |
| `GET stemwater` | `station-id` | `{hourly_values:[…]}`: the latest 300 **hourly** readings: `Timestamp`, `HlyAirTmp_Value` (°F), `HlyRelHum_Value`, `HlyVapPres_Value`, plus the Dynamic model's running state (`dynmod_inter_e`, `dynmod_xi`) |
| `GET pistachio` | `station-id`, `chill-year` | daily `grow_degr_hrs_r` (GDH, Richardson) and `units_utah` |
| `GET almond-hull-split-gdd90` | `station-id`, `fbd` (bloom date `YYYY-MM-DD`) | 91 daily `grow_degr_days_ss` values (single-sine GDD, °C) from bloom |
| `GET peach-harvest-gdh30` | `station-id`, `fbd` | 31 daily `grow_degr_hrs_a` values (GDH, Anderson) from bloom |
| `GET wildeye` | `siteID`, `startDate`, `endDate`, `variables` (comma list or `all`), `format` (`csv_wide`, `csv_long`, `json`) | 302 redirect to a file of raw Wildeye station data (air temp, rain, RH, solar, wind, …). Wildeye only. |

The data refreshes from CIMIS every 3 hours. For hourly CIMIS data older than the last 300 hours, use the [CIMIS API](https://et.water.ca.gov/), which needs a free app key, so register early.

## Gotchas

- `chill-year` must be the two-year form (`2025-2026`). If you pass `2025`, you get an **empty array with status 200**, not an error.
- Arrays aren't sorted. Sort by `Date` (or `Timestamp`) before you accumulate.
- Values are **daily amounts**, not running totals. Take a cumulative sum from the season start yourself.
- Season start: FNRIC starts on **Sept 1**, while many growers start counting in **November**. Pick one and label it.
- Rows where `n_hours` < 24 are missing station data. Decide whether to skip, fill, or flag those days.
- Latitude and longitude are strings like `"36º20'10N / 36.336222"`. Use the number after the `/`.
- The hull split GDD is in **°C**, while chill thresholds are in **°F**. Convert carefully.
- Inactive stations (`IsActive: false`) only have historical data.
- Check your numbers: the [chill calculator](https://fruitsandnuts.ucdavis.edu/chill-calculator) shows season-to-date totals for every station. If yours match, your math is right.

## Forecasting chill and GDD

A projection is **observed so far + forecast + historical average**. It uses the same formulas as above, applied to forecast temperatures instead of measured ones.

| Source | What it gives | Horizon | Key? |
| --- | --- | --- | --- |
| [Open-Meteo forecast](https://open-meteo.com/en/docs) | Hourly `temperature_2m` and daily min/max for any lat/lon; add `temperature_unit=fahrenheit` | **16 days** (384 hours) | No |
| [NWS API](https://www.weather.gov/documentation/services-web-api) | `/points/{lat},{lon}`, then the `forecastHourly` URL in the response, in °F | ~6.5 days (156 hours) | No, but send a `User-Agent` header from servers |
| [Open-Meteo historical](https://open-meteo.com/en/docs/historical-weather-api) | Past hourly temperature for any lat/lon | Back decades | No |
| FNRIC `chill-models-download` | Every past season's daily chill for the station | Back to the 1980s for older stations | No |

The method, step by step:

1. **Observed to date.** Cumulative sum of FNRIC daily values from Sept 1 (or Nov 1) through yesterday.
2. **Next 7–16 days.** Pull hourly forecast temps for the station's lat/lon. Apply the chill-hour rule to each hour, or the GDD formula to each day's min/max, and add the results to step 1.
3. **Beyond the forecast.** For each remaining calendar day up to the target date, use the **average of that same day across past seasons** ("climatology"), taken from `chill-models-download`.
4. **Show uncertainty.** Run step 3 with the coldest and warmest past seasons as well as the average, and plot the projection as a band, not a single line.
5. **Check it (stretch).** Re-run your method as if it were two weeks ago and compare against what the station actually recorded.

Going further, this is the heart of the CIMIS-gap challenge:

- **Microclimate correction.** Forecast grids miss local cold pockets. Compare the last 300 hours from `stemwater` with Open-Meteo historical temps for the same point, compute the average offset, and apply it to the forecast.
- **Dynamic model.** Portions depend on the model's running state, so you can't just add hours. Continue the calculation from the latest `dynmod_inter_e` and `dynmod_xi` in `stemwater`, or rerun the full season with forecast hours appended.
- **Daily-only data** (the easiest stretch idea): estimate hourly temps from daily min/max with a sine curve, count chill hours, and check the result against real hourly data.

## Starter code

**Python (pandas):** season-to-date chill, comparison with past years, and a 16-day projection.

```python
import pandas as pd, requests

BASE = "https://f5dby14cj5.execute-api.us-west-2.amazonaws.com/prod/"
STATION = "CIMIS-6"  # Davis

# Station coordinates
st = requests.get(BASE + "stations").json()["stations"]
s = next(x for x in st if x["StationID"] == STATION)
lat = float(s["HmsLatitude"].split("/")[1])
lon = float(s["HmsLongitude"].split("/")[1])

# 1. Full daily chill history, cumulative per season (season starts Sept 1)
df = pd.read_csv(f"{BASE}chill-models-download?station-id={STATION}", parse_dates=["Date"])
df = df.sort_values("Date")
df["season"] = df["Date"].dt.year.where(df["Date"].dt.month >= 9, df["Date"].dt.year - 1)
start = pd.to_datetime(df["season"].astype(str) + "-09-01")
df["day_of_season"] = (df["Date"] - start).dt.days
df["cum_32_45"] = df.groupby("season")["hours_32_45"].cumsum()

# 2. This season vs. past seasons at the same point in the season
latest = df["season"].max()
now = df[df.season == latest].iloc[-1]
past = df[(df.season < latest) & (df.day_of_season == now.day_of_season)]
print(f"{now.cum_32_45:.0f} chill hours (32-45F) so far; "
      f"past median at this date: {past.cum_32_45.median():.0f}")

# 3. Forecast: hourly temps from Open-Meteo, picking up after the last observed day
fc = requests.get("https://api.open-meteo.com/v1/forecast", params={
    "latitude": lat, "longitude": lon, "hourly": "temperature_2m",
    "temperature_unit": "fahrenheit", "past_days": 7, "forecast_days": 16,
    "timezone": "America/Los_Angeles"}).json()
temps = pd.Series(fc["hourly"]["temperature_2m"], index=pd.to_datetime(fc["hourly"]["time"]))
temps = temps[temps.index >= now.Date + pd.Timedelta(days=1)]
forecast_hours = ((temps > 32) & (temps <= 45)).sum()
print(f"Projected by {temps.index.max():%b %d}: {now.cum_32_45 + forecast_hours:.0f}")
```

**GDD, single sine** (a port of FNRIC's function; temperatures in °C):

```python
from math import asin, cos, pi

def gdd_single_sine(tmin, tmax, lo=5.0, hi=35.0):
    mean, a = (tmax + tmin) / 2, (tmax - tmin) / 2
    if tmin >= hi: return hi - lo
    if tmax <= lo: return 0.0
    if tmax <= hi and tmin >= lo: return mean - lo
    if tmax <= hi:  # tmin < lo
        s1 = asin((lo - mean) / a)
        return ((mean - lo) * (pi / 2 - s1) + a * cos(s1)) / pi
    if tmin >= lo:  # tmax > hi
        s2 = asin((hi - mean) / a)
        return ((mean - lo) * (s2 + pi / 2) + (hi - lo) * (pi / 2 - s2) - a * cos(s2)) / pi
    s1, s2 = asin((lo - mean) / a), asin((hi - mean) / a)
    return ((mean - lo) * (s2 - s1) + a * (cos(s1) - cos(s2)) + (hi - lo) * (pi / 2 - s2)) / pi
```

**JavaScript (browser or Node 18+):** this season's cumulative chill.

```javascript
const BASE = "https://f5dby14cj5.execute-api.us-west-2.amazonaws.com/prod/";
const url = new URL(BASE + "chill-models");
url.searchParams.set("station-id", "CIMIS-6");
url.searchParams.set("chill-year", "2026-2027");

const { model_values } = await (await fetch(url)).json();
model_values.sort((a, b) => a.Date.localeCompare(b.Date));
let total = 0;
const series = model_values.map(d => ({ date: d.Date, cum: (total += d.hours_32_45) }));
console.log(series.at(-1));
```

## Sources

- FNRIC: [chill calculator](https://fruitsandnuts.ucdavis.edu/chill-calculator), [about chilling hours, units, and portions](https://fruitsandnuts.ucdavis.edu/about-chilling-hours-units-and-portions), [almond hull split model](https://fruitsandnuts.ucdavis.edu/almond-hull-split-prediction-model), [station data](https://fruitsandnuts.ucdavis.edu/station-data), [all weather models](https://fruitsandnuts.ucdavis.edu/weather-models)
- [CIMIS](https://cimis.water.ca.gov/) (California Department of Water Resources weather stations)
- [Open-Meteo forecast API](https://open-meteo.com/en/docs), [Open-Meteo historical API](https://open-meteo.com/en/docs/historical-weather-api), [NWS API](https://www.weather.gov/documentation/services-web-api)
- GDD method: Zalom, F.G. et al. (1983), *Degree-days: the calculation and use of heat units in pest management*, UC Division of Agricultural Sciences Leaflet 21373
