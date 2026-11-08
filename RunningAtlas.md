# Prompt: build a personal "Running Atlas" from Garmin (paste everything below into Claude Code)

---

Build me a personal running dashboard called **Running Atlas** from my Garmin data, as a small local web app. Work in an empty folder. Use plain Node (18+, no npm dependencies) and a single-page vanilla HTML/JS frontend (no frameworks, no CDN). Everything stays on my machine.

## Data source: Garmin Connect *Personal API* (not the partner/OAuth program)
I created a personal API key at https://connect.garmin.com/app/settings/personalApiToken (docs: https://developer.garmin.com/gc-developer-program/personal/personal-api/). It is pull-based: **no webhooks, no OAuth, no tunnel.**

- Exchange the key for a 24h access token (JWT, no refresh tokens):
  `POST https://personal.connectapi.garmin.com/personalapitoken-service/personal-api/personal-api-key/token` with header `Authorization: ApiKey <key>`. Response has an access token (handle `access_token`/`accessToken`/`token`).
- All data calls go to host `https://personal.connectapi.garmin.com` with `Authorization: Bearer <token>`.
- `.env` holds `JWT_TOKEN` (optional, current access token) and `GARMIN_API_KEY` (so the app auto-renews the token when it expires or a call returns 401). Never print or log secrets. Load with `node --env-file-if-exists=.env`. Add `.env` and `data/` to `.gitignore`.
- **Sanitize the token:** strip whitespace, quotes and a trailing `%` (zsh's "no newline" marker people copy by accident; it makes Garmin answer "Bearer token is malformed").
- Endpoints that work:
  - List runs: `GET /activitylist-service/activities/search/activities?start=0&limit=100&activityType=running` (page with `start` until a page is shorter than `limit`; cap around 5000).
  - Per-run time series + GPS: `GET /activity-service/activity/{id}/details?maxChartSize=600&maxPolylineSize=600`. Response has `metricDescriptors` (`key`, `metricsIndex`) and `activityDetailMetrics[].metrics[]`; keys include `sumDistance`, `sumDuration`, `directHeartRate`, `directSpeed`, `directElevation`, `directDoubleCadence`; and `geoPolylineDTO.polyline[] {lat, lon}`. For the route atlas use `maxChartSize=5&maxPolylineSize=150` (small payload).
  - There is NO splits endpoint (404) and no HR-zone endpoint. Compute km splits from the time series by interpolation.
- Activity list field names (verify against a real response first): `activityId, activityName, startTimeGMT ("YYYY-MM-DD HH:MM:SS", UTC), distance (m), duration (s), averageHR, maxHR, elevationGain, calories, averageRunningCadenceInStepsPerMinute, fastestSplit_1000 / _1609 / _5000 (seconds), hrTimeInZone_1.._5 (seconds), minTemperature, maxTemperature, startLatitude, startLongitude, aerobicTrainingEffect, maxSpeed`.
- First **probe** the API with one request and look at the real JSON shape before writing the mapper. Ignore garbage `maxHR` values above ~215.

## Backend (server.js, bound to 127.0.0.1, port 3000)
- `lib/garmin.js` (token handling, `fetchRuns({full})`, `fetchDetail(id)`, `fetchRoute(id)`), `lib/store.js` (JSON file `data/runs.json`, upsert by id), `lib/crawl.js`.
- `npm run sync` (latest 30 runs) and `npm run sync -- --full` (entire history). Also a `POST /api/sync` for a Sync button.
- `GET /api/runs`, `GET /api/run/:id` (detail, cached in `data/details/`), `GET /api/routes`, and `POST/GET /api/crawl`: a background job (3 concurrent workers, retry on 429 with backoff, stop on 401, save every 25) that stores a compact polyline per run in `data/routes.json`. Runs with no GPS get `[]`.
- Static files from `public/`.
- Optional: `npm run import -- Activities.csv` for the Garmin Connect CSV export as an offline fallback, and a `seed-demo.js` that makes fake runs flagged `demo:true` so the UI can be previewed (real sync clears them).

## Frontend: `public/index.html` = the Atlas (dark, modern, single page, light/dark via `prefers-color-scheme`)
Use design tokens as CSS variables. One axis per chart (never dual-axis), thin marks, recessive grid, tooltips/crosshairs on every chart, text in ink colours (not series colours). Sections, top to bottom:

1. **Hero:** big total km, count-up tiles (km, runs, hours, avg km/run, avg pace), and a "journey" bar: straight-line distance from my home city to landmark cities (pick a few non-overlapping cities relative to where I live) with "X km to <next city>".
2. **Route atlas:** `<canvas>` heatmap of ALL routes drawn additively (`globalCompositeOperation='lighter'`, 3 passes: wide faint glow + 2 thin strokes) on a near-black background. Year filter chips, pan (pointer drag), wheel zoom around cursor, double-click reset, and a **Replay** button that animates runs chronologically over ~9s with a date + running total (newest ~12 runs in bright orange, source-over). Gotchas: project with `x=(lon-lon0)*cos(lat0), y=-(lat-lat0)`; **split each route wherever consecutive points jump > 0.02° (GPS glitches/flights) so no straight lines are drawn**; set the initial view from the ~60th percentile of point distances from the median start so the home area fills the canvas (outlier cities are reachable by zoom-out); show a progress overlay while the crawl runs and poll `/api/crawl`.
3. **"What your data says":** auto-generated insight cards computed from the data (longest break, current fitness vs peak, engine check, heat tax, rhythm, this year vs best year, race outlook).
4. **Life calendar:** rows = years, columns = 53 weeks, colour = weekly km (single-hue ramp, sqrt-scaled to the 95th percentile), year totals on the right, hover tooltip. Heading computes the number of years.
5. **Fitness / fatigue / form:** Banister model. Daily load = TRIMP (`minutes * hrr * 0.64 * e^(1.92*hrr)`, `hrr=(avgHR-rest)/(max-rest)`, fallback hrr 0.6 when no HR). Fitness = EWMA 42d, fatigue = EWMA 7d, form = fitness − fatigue (separate smaller panel under it with a zero line). Resting-HR and max-HR inputs (defaults 55 and the 98th percentile of maxHR; saved in localStorage); range buttons 6m/1y/3y/All. The section title is a computed finding (e.g. "Fitness is at 20% of your peak").
6. **Aerobic efficiency:** metres per heartbeat (`distance / (minutes * avgHR)`) on steady runs (≥3 km, ≥20 min, avg HR 110–170, sane pace), monthly median with a 3-month rolling median; title states % change vs the previous 12 months.
7. **Heat tax:** scatter of pace vs temperature ((min+max)/2) for 3–25 km runs, least-squares line, label "+N s/km per 10°C", caveat that it is a raw correlation; title is the computed finding.
8. **Rhythm:** a 24-hour polar bar clock (peak hour highlighted) and a weekday bar chart; title says morning/afternoon/evening runner.
9. **Year race:** cumulative km by day-of-year, one thin muted line per year, best year and current year highlighted with direct end labels; y-axis max rounded to a multiple of 400 so ticks are clean; title says how far ahead/behind the best year at this point.
10. **Race predictor + hall of fame:** Riegel (`t2 = t1*(d2/d1)^1.06`) from the fastest 5 km split, "all-time" vs "last 12 months" for 5K/10K/half/marathon; record cards (fastest 1K, mile, longest run, biggest climb, biggest week, biggest month, longest streak, count of 21K+ and 10K+ runs).

Also keep a secondary page `public/classic.html` (linked from the Atlas header): range buttons (30d/90d/1y/All), summary tiles, weekly/monthly distance, pace trend and HR charts, HR-zone time bars, personal bests, and a clickable runs table that opens a **run detail** view via `#run/<id>` (stats, GPS route as an SVG polyline with start/end dots, pace/HR/elevation charts vs distance, km-splits table with the fastest km marked).

## Process
- Probe the API first, then build backend, then run `npm run sync -- --full` and start the route crawl.
- Verify in the browser: no console errors, every section renders with real data, check the atlas visually (zoom, year filter, replay). Fix layout problems you actually see (label collisions, clipped labels, over-zoomed map) instead of assuming.
- Finish by telling me how to refresh my token daily (or add `GARMIN_API_KEY` for auto-renew) and what each section shows.

My details: [PUT YOUR GARMIN API KEY IN .env YOURSELF; do not paste it in chat]. Home city for the journey bar: [YOUR CITY]. Units: [km / miles].
