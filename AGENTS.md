# Flight Cancel Predictor — Agent Context

## Project Overview

A single-file, zero-dependency browser app that predicts flight cancellation risk as a percentage (0–100%). The app combines airline historical risk, live weather data from Open-Meteo, departure time heuristics, route congestion, and flight distance into a weighted scoring formula.

**File:** `index.html` — all HTML, CSS, and JavaScript in one file. No build step, no server required. Open directly in a browser.

**Data files (JSON):**
- `airlines.json` — 20 US airlines, each with `code`, `name`, `riskScore` (1–10 scale)
- `airports.json` — ~100+ US airports, each with `code`, `name`, `city`, `state`, `latitude`, `longitude`, `severity` (a dictionary keyed by weather condition type with severity values)

## Architecture

### Structure
```
index.html          # Single-file SPA (HTML + <style> + <script>)
airlines.json       # Airline database
airports.json       # Airport database with weather severity profiles
README.md           # User-facing documentation
```

### Key Globals (script scope)
| Variable | Type | Description |
|----------|------|-------------|
| `airports` | `Airport[]` | Populated from `airports.json` (deduplicated by code) |
| `airlines` | `Airline[]` | Populated from `airlines.json` |
| `selectedAirline` | `Airline\|null` | Currently selected airline |
| `selectedDepAirport` | `Airport\|null` | Currently selected departure airport |
| `selectedArrAirport` | `Airport\|null` | Currently selected arrival airport |
| `airportsLoaded` | `boolean` | Whether airports.json has been fetched |
| `airlinesLoaded` | `boolean` | Whether airlines.json has been fetched |

### Core Functions (in order of execution)

#### Prediction Pipeline
1. **`predict(airline, depAirport, arrAirport, departureTimeMinutes)`** — Top-level orchestrator. Calls `calculateWeatherRisk`, computes airline/time/route/distance risk, applies weighted sum, clamps 0–100, calls `displayResult`.
2. **`calculateWeatherRisk(depAirport, arrAirport, departureTimeMinutes)`** — Fetches live weather via Open-Meteo at departure airport (current conditions) and arrival airport (estimated arrival time). Returns `{ score, conditions[], detailScores, liveData, flightTime, arrivalTimeMinutes }`.
3. **`fetchWeatherCurrent(lat, lon)`** — GET `https://api.open-meteo.com/v1/forecast?latitude=...&longitude=...&current=weather_code,wind_speed_10m&timezone=auto`
4. **`fetchWeatherAtTime(lat, lon, targetMinutesSinceMidnight)`** — Fetches hourly forecast, finds closest hour to the target time. Returns `{ code, wind }` or `null`.
5. **`evaluateWeatherConditions(code, windSpeed)`** — Maps WMO weather code → condition type + base score via `wmoToConditions()`, applies severity weights from `severityWeights` map, returns `{ score, conditions[] }`.
6. **`wmoToConditions(code, windSpeed)`** — WMO code → condition mapping. Returns arrays of `{ condition, score }` including weather code conditions and wind-based conditions.
7. **`estimateFlightTimeMinutes(distanceKm)`** — Threshold-based: <300→75, <800→120, <1500→180, <2500→240, else 360 (minutes).
8. **`haversine(lat1, lon1, lat2, lon2)`** — Standard haversine distance in km.

#### Weather Scoring (inside `calculateWeatherRisk`)
```
depScore = evaluate(depWeatherCode, depWind).score / 20
arrScore = evaluate(arrWeatherCode, arrWind).score / 20
timeMultiplier: if departureHour 0–5 → depScore *= 1.3; if 19–23 → depScore *= 1.15
weatherScore = (depScore * 0.5 + arrScore * 0.5) * 20
compoundMultiplier applied when both airports have: ice×2 → 1.4, snow×2 → 1.3, thunderstorm×2 → 1.25, fog×2 → 1.2
final weatherScore *= compoundMultiplier
```

#### Main Prediction Formula (inside `predict`)
```
weatherRiskScore    → weight 0.45
airlineRiskScore    → weight 0.15  (airline.riskScore * 1.5)
timeRisk            → weight 1.0   (12 if 0–5h, 8 if 19–23h, 6 if 11–13h, 4 otherwise)
routeRisk           → weight 1.0   (10 if both endpoints are major hubs, 4 otherwise)
distanceFactor      → weight 1.0   (8 if different states, 5 otherwise)
random jitter       → [-3, +3]
```
Result clamped to `[0, 100]` and rounded.

#### UI Functions
- **`setupAutocomplete(inputId, listId, data, displayFields, valueFormatter, onSelect)`** — Generic autocomplete: debounced 80ms input filtering, keyboard nav (ArrowUp/Down/Enter/Escape), click selection. Highlights matching text.
- **`initAutocomplete()`** — Initializes three autocomplete instances: airlines (`name`, `code`), departure airports (`code`, `city`, `state`), arrival airports (`code`, `city`, `state`).
- **`displayResult(pct, data)`** — Renders result card. Color thresholds: ≤30 → green/LOW, 30–60 → amber/MODERATE, ≥60 → red/HIGH. Also sets page background color.
- **`minutesToLabel(m)`** — Converts minutes since midnight to `h:00 AM/PM` string.

#### Save/Load/Clear (localStorage)
- **Key:** `flightPredictor_data`
- **Schema:** `{ airline, airlineCode, depAirport, depCode, arrAirport, arrCode, departureTime, savedAt }`
- **`saveFormToStorage()`** — Writes current form state to localStorage.
- **`loadFormFromStorage()`** — Reads from localStorage, matches by code to populate globals, fills form inputs, re-enables predict button if all fields present. Auto-called on `DOMContentLoaded`.
- **`clearFormAndStorage()`** — Removes localStorage entry, clears all form fields and globals, disables predict button.

### HTML Elements by ID
| ID | Element | Purpose |
|----|---------|---------|
| `airlineInput` | input[text] | Airline search input |
| `airlineList` | div | Autocomplete dropdown for airlines |
| `depAirportInput` | input[text] | Departure airport search |
| `depAirportList` | div | Autocomplete dropdown |
| `arrAirportInput` | input[text] | Arrival airport search |
| `arrAirportList` | div | Autocomplete dropdown |
| `departureTime` | select | Time slot dropdown (24h × 4 per day = 96 options, 15-min intervals) |
| `predictBtn` | button | Triggers prediction (disabled until all 3 fields + weather ready) |
| `saveBtn` | button | Saves form to localStorage |
| `loadBtn` | button | Loads form from localStorage |
| `clearBtn` | button | Clears form and localStorage |
| `result` | div | Result card (hidden until prediction) |
| `percentage` | div | Cancellation percentage display |
| `statusText` | div | Risk level label (LOW/MODERATE/HIGH) |
| `detailBar` | div | Detailed breakdown rows |
| `loading` | div | Loading/notification overlay |
| `background-layer` | div | Background color layer |

### Autocomplete Data Display Formats
- **Airline:** `valueFormatter` = `item.name` (displays full name, e.g., "Delta Air Lines")
- **Airport:** `valueFormatter` = `` `${item.code} - ${item.city}, ${item.state}` `` (e.g., "ATL - Atlanta, GA")

## Data Models

### Airline (`airlines.json` entry)
```json
{
  "code": "DL",
  "name": "Delta Air Lines",
  "riskScore": 3
}
```
- `riskScore`: integer 1–10. Higher = more cancellations historically.

### Airport (`airports.json` entry)
```json
{
  "code": "ATL",
  "name": "Hartsfield-Jackson Atlanta International Airport",
  "city": "Atlanta",
  "state": "GA",
  "latitude": 33.6407,
  "longitude": -84.4277,
  "severity": {
    "fog": 5,
    "ice": 6,
    "snow": 4,
    "thunderstorm": 12,
    "wind": 5,
    "haze": 4,
    "rain": 8,
    "heavy_snow": 3,
    "heavy_rain": 7
  }
}
```
- `severity`: keys are condition types, values are severity scores (higher = worse impact on cancellations for this airport).
- Not all airports have coordinates (some minor airports lack `latitude`/`longitude`).

## Style Rules

- **Zero dependencies** — no frameworks, no libraries. Vanilla JS only, no bundling.
- **Single-file architecture** — everything lives in `index.html`. Do not split modules.
- **Inline CSS** — all styles in `<style>` block within `<head>`.
- **External assets** — only `airlines.json` and `airports.json` are fetched at runtime via `fetch()`.
- **Weather API** — only uses `api.open-meteo.com` endpoints. Fallback paths exist when weather data is unavailable (graceful degradation to default scores).
- **No external CSS/JS files** — any additions should stay within the existing `<style>` and `<script>` blocks.
- **CSS class naming** — uses BEM-ish pattern: `.container`, `.form-group`, `.detail-row`, `.autocomplete-item`. Background color transitions on `#background-layer`.
- **No tests framework** — this is a static single-file app with no test infrastructure.

## Constraints and Conventions

- The `predictBtn` is disabled until all three selections (airline, dep airport, arr airport) are made AND both JSON files are loaded.
- Weather fetch happens on every prediction call (not cached). Timeout: 5s default from browser.
- `localStorage` key is exactly `flightPredictor_data`.
- Time values are minutes-since-midnight integers. The `<select>` uses values `0, 15, 30, ..., 1425` (covering all 96 15-min slots in a 24h day).
- The `majorHubs` Set in `predict()` is hardcoded and contains many airport codes (some duplicates present — do not deduplicate unless explicitly asked).
- Random jitter is `(Math.random() * 6 - 3)` — this is intentional for slight variance.
- When weather API fails, the function returns `null`, and a default weather score of 2 is used (lowest risk).
- The `checkLoads()` function gates autocomplete initialization until both JSON fetches complete.
