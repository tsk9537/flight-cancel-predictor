# Improvement Suggestions

Comprehensive list of improvements for the Flight Cancellation Predictor project, organized by priority and category.

---

## Critical Bugs

### 1. `fetchWeatherAtTime` — `bestIndex` always set to `0` (line 571)

The first search loop sets `bestIndex = 0` instead of `bestIndex = i`, meaning it never captures the actual closest forecast hour. It effectively discards the first loop's work and relies entirely on a second loop that starts from an already-broken state.

```js
// Current (broken)
if (diff < bestDiff) { bestDiff = diff; bestIndex = 0; }

// Should be
if (diff < bestDiff) { bestDiff = diff; bestIndex = i; }
```

This causes the weather lookup to frequently return the wrong hourly forecast slot, degrading prediction accuracy.

### 2. `fetchWeatherAtTime` — Duplicated / conflicting double-loop search (lines 566–580)

Two sequential loops attempt to find the closest match. The second loop recomputes the closest time, negating the first loop entirely. Consolidate into a single pass.

### 3. `fetchWeatherCode` — `bestIndex` never updated from `0` (line 482)

Same bug as #1. The `findIndex` callback sets `bestIndex = 0` in its side effect, but `findIndex` uses the return value for logic. The loop variable `i` is never used after the assignment, making the whole block inert.

### 4. `minutesToLabel` — `mins` hardcoded to `0` (line 737)

```js
const mins = 0;  // Always zero — no minutes displayed
```

This should use `m % 60` to support the 15-minute interval time selections in the UI. Currently all times render as ":00" regardless of which slot is selected.

### 5. `minutesToLabel` — `hrs %= 24` strips next-day hours (line 736)

When departure time + flight time exceeds midnight (e.g., 25:00 = 1:00 AM next day), `m % 24` silently wraps to 1:00 AM. The `+1h` suffix in the arrival display is a partial fix but should be explicit (e.g., "1:00 AM (+1d)").

### 6. `fetchWeatherAtTime` — fallback `bestIndex = 0` silently (line 582)

If `bestIndex` exceeds the array bounds, the code falls back to index `0` (the first hour of the first forecast day) instead of returning `null` or reporting the error.

---

## High Priority

### 7. Major hub list typo: `LAZ` instead of `LAX` (line 709)

`LAZ` is not a valid airport code (Las Vegas is `LAS`, Los Angeles is `LAX`). Both `LAX` and `LAS` appear in the set but `LAX` is listed twice while `LAZ` is a non-existent code. Deduplicate and fix the typo.

### 8. Hardcoded timezone in `fetchWeatherAtTime` (line 553)

Uses the user's browser timezone for the forecast query. While Open-Meteo accepts `timezone: auto`, using the user's local timezone to parse arrival times can produce incorrect results when the user is in a different timezone than the airports being queried. The arrival time should be resolved relative to the arrival airport's timezone.

### 9. No error handling for weather API network failures

When the weather API is unreachable or returns an error response (HTTP ≥ 400), `resp.json()` can throw or return an unexpected structure. The `try/catch` catches the error but silently falls back to a weather score of `2`, providing no indication to the user that the prediction is based on defaults rather than live data.

---

## Medium Priority

### 10. No caching of weather API responses

Every prediction call fetches fresh weather data, even for identical queries entered multiple times. Implement an in-memory cache with TTL (e.g., 5–15 minutes) keyed by airport code + estimated arrival time. This reduces API load and provides consistent results across repeated predictions.

### 11. `fetchWeatherAtTime` never uses the `targetDate` variable (line 550)

A `targetDate` is computed on line 550 but never referenced in the subsequent loops. The loops operate on raw timestamps instead, making the date parsing indirect and error-prone.

### 12. No date input — predictions use implicit "today"

Users select only a departure time slot (minutes-since-midnight) but cannot specify the date. If the user selects a time 5 hours from now, the code computes offset from the current moment, but weather forecasts from Open-Meteo may not cover that far. The app silently falls back to current conditions rather than using a proper future date.

### 13. Predict button can be re-clicked during an in-flight prediction

The `predictBtn` is not disabled during the async prediction, allowing multiple simultaneous weather fetches. The user sees multiple loading spinners and potentially stale results.

### 14. Autocomplete lacks keyboard accessibility standards

The autocomplete list uses `mousedown` events rather than `click`, and does not set `role="listbox"` / `role="option"`, `aria-activedescendant`, or `aria-expanded` on the input. This makes the form unusable with screen readers and keyboard-only navigation.

### 15. `routeRisk` has two possible values (4 vs 10) but no gradient

Major hub-to-major hub routes always get the maximum route risk of 10, while everything else gets 4. There is no middle ground (e.g., one hub + one non-hub). The major hubs set also lacks distinction between primary (e.g., ATL, LAX) and secondary hubs.

### 16. Time-based risk uses broad 3-hour windows

```js
if (departureHour >= 0 && departureHour <= 5) timeRisk = 12;   // 6 hours, risk 12
else if (departureHour >= 19 && departureHour <= 23) timeRisk = 8;
else if (departureHour >= 11 && departureHour <= 13) timeRisk = 6;
```

The late-night slot (midnight–5am) gets the highest penalty with a 6-hour wide range. A more granular model (e.g., 0–2h = 14, 2–5h = 10, 5–8h = 4) would better reflect actual historical cancellation patterns.

### 17. Compound weather multiplier triggers on condition type overlap only

The compound multipliers (lines 666–674) fire when the *type* of condition is the same at both endpoints (e.g., both have "ice"). This ignores severity: a minor fog at both airports triggers the same 1.2x multiplier as heavy fog at both. A severity-based threshold would be more accurate.

### 18. Weather severity data missing coordinates on ~10 airports

Airports like RAP, CYS, COU, MDW, SBN, SDF, TWF, BTR, TYS, CHS, SAV, RSW, SRQ, PNS, IWA have `severity` profiles but lack `latitude`/`longitude`. These airports silently skip weather fetching (a graceful degradation), but users may not realize the prediction is based on defaults for the weather factor.

---

## Low Priority — Feature Gaps

### 19. No result sharing / export

Users cannot copy a prediction result, share it via URL, or export to a file. Consider adding a "Copy Result" button that generates a shareable URL with query parameters encoding the flight selection.

### 20. No prediction history / past results

The app does not track previous predictions. A simple in-app history list (stored in localStorage) would let users compare predictions over time.

### 21. No "Add to Calendar" or notification feature

After a prediction, users could add the flight to their calendar. This is a stretch goal but would improve integration.

### 22. No dark mode

The current UI is light-only. A dark mode toggle would improve usability and is straightforward to add with a CSS class toggle on `body`.

### 23. No mobile responsiveness testing

The layout uses `max-width: 560px` and flexbox, which generally adapts, but the autocomplete dropdowns may overflow on narrow screens. Test and fix for 320px–480px widths.

### 24. No input validation feedback

If a user enters an airline name but selects nothing by the time they click predict, the click handler silently does nothing (line 804). Providing visible feedback ("Please select all fields") would be better.

### 25. No loading / progress indicator during weather API calls

The loading overlay shows a spinner but the text says "Calculating..." then changes to "Fetching live weather." A more granular progress indicator would communicate that two weather fetches are happening sequentially.

---

## Low Priority — Data & Algorithmic Improvements

### 26. Airline risk scores are static (not time-varying)

The `riskScore` in `airlines.json` is a single static integer. Real-world airline reliability varies by season, year, and region. Consider adding a time-varying field or fetching historical performance data.

### 27. No flight status / delay data integration

The predictor does not integrate with flight-status APIs (e.g., FlightAware, FlightRadar24). Adding a delay probability factor from current airport traffic would improve accuracy.

### 28. Distance factor uses only state boundary, not distance magnitude

Routes crossing state boundaries get 8, same-distance intra-state gets 5. There is no gradient based on actual distance (e.g., NY→FL = 10, NY→Boston = 5). The haversine distance is only used for flight time estimation, not the prediction formula.

### 29. Route congestion only considers binary hub/non-hub

A more sophisticated model could weight the number of carriers operating the route, airport congestion levels, and historical delay rates at that route. The current binary major-hub check is a reasonable approximation but could be refined.

### 30. No consideration of day of week / seasonality

Flight cancellation patterns differ significantly by day of week (Fridays and Sundays have higher delay rates) and season (winter has higher weather-related cancellations). A date-aware adjustment could be beneficial.

---

## Architecture & Code Quality

### 31. Single-file architecture blocks testing

All code lives in one 930-line `<script>` block. Extract the prediction logic (`predict`, `calculateWeatherRisk`, weather evaluations) into a separate module so it can be unit-tested with Jest or a similar framework.

### 32. No JSDoc or inline documentation in JS

The JavaScript functions have no JSDoc comments. Adding type signatures and descriptions would help future maintainers.

### 33. Hardcoded magic numbers throughout

Constants such as `0.45`, `0.15`, `1.3`, `1.15`, `1.2`, `20`, `3`, `6`, `1425` appear without explanation. Extract them into a named constants object at the top of the script.

### 34. No separation of concerns in `fetchWeatherAtTime`

This function handles: timezone resolution, API querying, time-slot matching, bounds checking, error handling, and data extraction all in one 48-line block. Split into smaller functions: `buildForecastUrl`, `findClosestForecastSlot`, `parseForecastResponse`.

### 35. No telemetry / error reporting

Failed weather fetches and prediction errors only go to `console.warn`. In a production setting, a telemetry endpoint or error reporting integration would help identify user-side failures (e.g., blocked ad-blockers blocking the API).

### 36. `localStorage` has no schema versioning

If the saved form schema changes, there is no migration path for old stored data. Adding a version field to the stored JSON enables graceful migration.

### 37. CSS lacks a design system / custom properties

All colors, spacing, and font sizes are hardcoded. Convert to CSS custom properties (variables) for easy theming, dark mode support, and consistency.

### 38. `setupAutocomplete` — no way to select "no results" option

When the autocomplete filter returns zero matches, the user has no visible way to confirm that no result exists. A "No results found" message in the dropdown would improve UX.

### 39. Predict button text changes inconsistently

The button text cycles through: "Loading data..." → "Predict Cancellation Risk" → "Fetching live weather..." → back. The "Loading data..." text uses the initial state to indicate loading, which could confuse users. Use a separate loading state indicator.

---

## Quick Wins (Easy to Implement)

| # | Change | Effort |
|---|--------|--------|
| 1 | Add `mins = m % 60` in `minutesToLabel` | Low |
| 2 | Fix `bestIndex = i` in `fetchWeatherAtTime` | Low |
| 3 | Fix `bestIndex = i` in `fetchWeatherCode` | Low |
| 4 | Deduplicate `majorHubs` set and fix `LAZ` → `LAX` | Low |
| 5 | Disable predict button during prediction | Low |
| 6 | Add user-agent-friendly error message on failed weather fetch | Low |
| 7 | Extract magic numbers into named constants | Medium |
| 8 | Add JSDoc comments to public functions | Medium |
| 9 | Add "No results" message to autocomplete | Low |
| 10 | Add CSS custom properties for theming | Medium |

---

## Suggested File Restructure (Future)

Once the app grows, consider splitting the monolith:

```
flight-cancel-predictor/
├── src/
│   ├── index.html           # Shell HTML, links to modules
│   ├── js/
│   │   ├── prediction.js    # predict(), calculateWeatherRisk(), scoring logic
│   │   ├── weather.js       # fetchWeatherCurrent, fetchWeatherAtTime, wmoToConditions
│   │   ├── ui.js            # Autocomplete, displayResult, DOM manipulation
│   │   ├── storage.js       # Save/Load/Clear, localStorage handling
│   │   └── helpers.js       # haversine, minutesToLabel, escapeHtml
│   ├── css/
│   │   └── main.css         # All styles (extract from <style>)
│   └── test/
│       ├── test-prediction.js
│       ├── test-weather.js
│       └── test-ui.js
├── data/
│   ├── airlines.json
│   └── airports.json
└── README.md, AGENTS.md
```

This structure enables independent testing of the prediction algorithm and weather parsing, separate from the DOM-based UI layer.
