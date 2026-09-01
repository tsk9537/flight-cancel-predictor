# Flight Cancellation Predictor

A browser-based flight cancellation risk predictor. Select an airline, departure/arrival airports, and departure time to get a real-time cancellation risk percentage powered by live weather data.

## Features

- **Real-time weather integration** — Fetches current and forecasted weather from [Open-Meteo API](https://open-meteo.com/) at both departure and estimated arrival times.
- **Weighted risk scoring** — Cancellation probability is calculated from five factors: weather severity (45%), airline historical risk (15%), departure time slot (20%), route hub congestion (10%), and flight distance (5%).
- **Airport-specific weather susceptibility** — Each airport in the dataset carries a severity profile (e.g., ORD has high snow/wind scores, SFO has high fog scores) that modulates the weather risk.
- **Compound weather multipliers** — When multiple severe conditions occur at both ends (ice + snow, thunderstorm at both airports, etc.), risk scores are amplified.
- **Autocomplete search** — Type-ahead selection for 160+ US airports with code, city, and state display, and 20 US airlines.
- **Save / Load / Clear** — Persist form selections in `localStorage` so your inputs survive page refreshes.

## Data Sources

| Source | File | Description |
|--------|------|-------------|
| Airlines | `airlines.json` | 20 US airlines with IATA codes and historical risk scores (1–10 scale) |
| Airports | `airports.json` | ~100+ US airports with coordinates, city/state, and weather severity profiles per condition type |
| Weather | [Open-Meteo API](https://open-meteo.com/) | Hourly weather codes, wind speed, and temperature for real-time conditions |

## How It Works

The prediction formula combines these weighted factors:

```
cancellationPct = weatherRisk * 0.45
              + airlineRisk * 0.15
              + timeRisk * 1.0
              + routeRisk
              + distanceFactor
              + random jitter (-3 to +3)
```

### Weather Scoring

1. WMO weather codes are mapped to condition types with base weights (thunderstorm: 20, heavy snow: 20, ice: 18, heavy rain: 14, etc.)
2. Each airport has condition-specific severity modifiers (e.g., DEN snow severity = 16)
3. A compound multiplier amplifies risk when multiple severe conditions exist at both departure and arrival
4. Early morning departures (midnight–5 AM) receive a 1.3x penalty on departure weather

### Runtime Setup

1. Open `index.html` in a browser directly (double-click the file or use `open index.html`)
2. No build step or server required — it's a static HTML file that fetches JSON data from the same directory

## File Structure

```
flight-cancel-predictor/
├── index.html       # Single-page app (UI + prediction logic)
├── airlines.json    # Airline data with risk scores
└── airports.json    # Airport data with severity profiles
```

## Browser Requirements

- Modern browser with ES6+ and `fetch` API support (Chrome 90+, Firefox 88+, Safari 14+, Edge 90+)
