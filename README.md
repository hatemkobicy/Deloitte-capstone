# Powering Ahead — Grid Stress Monitor NL

Prototype exploratory dashboard for the Deloitte capstone *"Predicting Local Grid Stress in the Netherlands."*
The page (`index.html`) is a static, single-file dashboard. It reads all of its numbers from **`data.json`**
in this same repo — replace that file with your model team's real output and the dashboard updates
automatically, with no code changes.

## 1. How the dashboard gets its data

On load, `index.html` does:

```js
fetch('./data.json')
```

- If `data.json` exists and has at least one entry in `buurten`, the dashboard shows a **"Live model
  data"** badge and renders it.
- If the file is missing, empty, or fails to parse, it silently falls back to a **built-in demo
  dataset** (synthetic, seeded) and shows a **"Demo data"** badge — so the page never breaks, even
  before the model is ready.

You (dashboard owner) don't need to touch the JS for this to work. Whoever owns the model just needs
to keep `data.json` updated in this repo, matching the schema below.

## 2. Data contract — what `data.json` must contain

Minimum required: **`buurten`** — a list of area records. Everything else is optional; if omitted,
the dashboard computes reasonable fallbacks so partial submissions still render.

```jsonc
{
  "kpis": {                                   // optional — auto-computed from buurten if omitted
    "national_peak_demand_gw": 17.4,
    "transformers_over_nbl": 681,
    "transformers_total": 9412,
    "avg_utilization_pct": 61,
    "ev_penetration_pct": 9.8
  },
  "demand_trend": {                            // optional — used in "Demand trends" tab
    "labels": ["2019-01", "2019-02", "..."],   // any string labels, one per point
    "values_mw": [812, 795, "..."]             // demand in MW (or an index if not yet calibrated)
  },
  "generation_mix": {                          // optional
    "labels": ["Fossil gas", "Wind onshore", "Solar", "Wind offshore", "Nuclear", "Other"],
    "values_pct": [29, 21, 19, 14, 8, 9]        // must sum to ~100
  },
  "province_demand": {                         // optional
    "labels": ["Flevoland", "Fryslân", "..."],
    "values_kwh": [2800, 3100, "..."]           // kWh per residential connection per year
  },
  "seasonal": {                                // optional — used in "Seasonal patterns" tab
    "months": ["Jan","Feb","...","Dec"],
    "demand_index": [68, 62, "..."],
    "hdd_celsius": [210, 180, "..."]            // Heating Degree Days, Celsius-based (never °F)
  },
  "buurten": [                                  // REQUIRED — at least one entry
    {
      "name": "Kraggenburg",
      "province": "Flevoland",
      "utilization_pct": 61.2,                  // observed peak load ÷ NBL threshold × 100
      "ev_pct": 8.4,                             // EV share of local car fleet, %
      "pv_pct": 22.1,                            // % of connections with solar PV
      "hdd_celsius": 1450,                       // annual heating degree days
      "housing_build_year_median": 1965,
      "population_density_per_km2": 900,

      "risk_score": 47.3,                        // optional 0–100; auto-computed if omitted
      "risk_drivers": {                          // optional; auto-computed if omitted
        "transformer_utilization": 55,
        "ev_density": 27,
        "heating_degree_days": 21,
        "solar_pv_share": 18,
        "housing_age": 40,
        "population_density": 20
      }
    }
  ]
}
```

**Units, always:**
- Temperature-derived fields (`hdd_celsius`) are Celsius-based — KNMI never reports Fahrenheit.
- `utilization_pct` is a percentage of Liander's **NBL** operational threshold, not raw nameplate capacity — values can legitimately exceed 100.
- `risk_score` and every value inside `risk_drivers` are **unitless**, rescaled onto a comparable 0–100-ish range so factors with different native units can be ranked together.

A minimal valid file is just:
```json
{ "buurten": [ { "name": "Test", "province": "Flevoland", "utilization_pct": 80, "ev_pct": 6, "pv_pct": 20 } ] }
```

## 3. Two ways the model team can deliver data

**Option A — commit a file (recommended for this project's timeline).**
The model team's notebook/script writes `data.json` in this exact shape and pushes it to the repo
(or opens a PR). Simplest, no server, no CORS to configure, works the moment GitHub Pages rebuilds
(usually under a minute).

**Option B — point at a live API.**
If they stand up an endpoint instead, open `index.html` and change one line near the top of the
`<script>` block:
```js
const DATA_URL = 'https://your-api.example.com/grid-stress-data';
```
Their API must (a) serve the exact JSON shape above and (b) return the header
`Access-Control-Allow-Origin: *` (or your Pages origin specifically) — otherwise browsers will block
the request as a CORS violation and the dashboard will silently fall back to demo data.

6. Share that link. Every future push to `main` (including a new `data.json` from your teammates)
   redeploys automatically within about a minute.

## 5. Local preview before pushing

No build step needed — just open `index.html` directly in a browser, or serve the folder locally:
```bash
python3 -m http.server 8000
# then visit http://localhost:8000
```
(A plain `file://` open also works for this page, but a local server is closer to how GitHub Pages
serves it.)
