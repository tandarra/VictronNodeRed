# Changelog

Newest first. Each entry = one commit + tag; `git show <tag>:flows.json` restores that version.

## v16.7 - 2026-11-06
**21:00 target never below 43%, and the big-day export stops quickly when a storm arrives**
- Problem 1: `forecastMinTarget` was 32, so a high forecast could pull the 21:00 target below the 43% base. 43% is the absolute bottom in any condition. `CONFIG.forecastMinTarget` is now 43 (and the Forecast Adjuster fallback), so the forecast can no longer lower the target; weather only raises it: dry 43, rain 49, heavy rain 53, thunderstorm 57.
- Problem 2: the v16.4 big-day latch kept exporting at -9000 W (mode NORMAL, SoC under the floor) after a thunderstorm arrived and PV collapsed, draining the battery to about 50% with almost no PV left to refill it. The latch only released after PV stayed under 3 kW for 45 minutes and ignored the weather.
- Fix: when a fresh `weatherImpact` shows heavy rain or a thunderstorm in the remaining daylight (severity >= 2), the low-PV timeout is `pvBigDayStormLowPvMin` (5 min, new CONFIG key) instead of `pvBigDayLowPvMin` (45 min). Ordinary rain and missing/stale weather data keep the 45 min behaviour. It still re-arms if PV climbs back over 8000 W.
- Tested offline: storm and heavy rain with PV 1 kW for 6 min release the latch (-9000 -> -150 W); 2 min, rain only, no weather data and PV above 3 kW keep exporting; no weather data releases at 46 min as before.

## v16.6 - 2026-11-06
**Rain and thunderstorms now change the controller's behaviour**
- Problem: the forecast only fed the controller as a kWh number. The 21:00 target adjustment (`min(forecast% x 0.6, 25)`) was already at its 25-point cap at 17.4 kWh, so a thunderstorm day got the same 32% target as any other day and nothing branched on the weather. The parser also used the DAILY weather code (worst of the whole day), so a storm at 3am labelled the whole day.
- New node `Weather impact (rain/storm)` between `Open-Meteo API` and `Forecast parser`. It reads the HOURLY weather codes, precipitation (mm) and probability for the remaining daylight hours (now to sunset) and writes `flow.weatherImpact`. The message passes through unchanged, so the parser is untouched and `flow.weather` is not overwritten.
- Severity: rain (codes 51-57/61/63/66/80/81 at >= 50% probability, or >= 0.5 mm/h), heavy rain (65/67/82 or >= 4 mm/h), thunderstorm (95/96/99).
- `Forecast Adjuster`: weekday 21:00 target +6 / +10 / +14 and daytime floor +0 / +3 / +5 (floor capped at 95). The floor bump applies on weekends too. Ignored if the forecast is invalid or the impact is older than 3 h (covers the `EAI_AGAIN` DNS failures). The PV forecast is deliberately NOT cut again: the POA radiation data already reflects cloud.
- `Build Open-Meteo URL`: also requests hourly `precipitation`.
- New CONFIG block `weatherImpact` (all bumps and thresholds in one place). `State publisher` adds `state.weatherImpact`. New node added to group 7 and to the forecast error catch scope. Tab label and CONFIG header bumped to v16.6.
- Node IDs in `flows.json` now match the live Cerbo flow (a re-import had regenerated them), so Import > Replace updates the existing tab instead of duplicating it.
- Tested offline: dry, rain, heavy rain and thunderstorm later today, storm already passed, stale (4 h) impact, invalid forecast, weekend, malformed payload. Thunderstorm on a 17.4 kWh day gives target 46% / floor 85%; rain gives 38% / 80%.

## Auto-merge - 2026-10-30
- `.github/workflows/sync-main.yml`: pushes to `claude/*` branches are merged into `main` automatically (no PR needed).

## Live page - 2026-10-30
- `docs/index.html`: GitHub Pages page embedding the VRM live view (Pages source: `main`, folder `/docs`). README links to it.

## v16.5 docs - 2026-10-30
- README rewritten with hero images (`docs/`), VRM live link and system overview.
- `homeassistant/`: pool controller documentation, the load-shed automation and the list of changes to `packages/pool_solar.yaml` (PV-surplus bypass, late run, load shed).

## v16.5 - 2026-10-30
**Fix: daily energy counters wiped on redeploy**
- Problem: after a redeploy (11:57 today) `ESS Grid Export Today` and `ESS AC Consumption Today` restarted from 0 (HA card showed 3.7 kWh while the meter and VRM showed 19.7 kWh). The `Energy snapshot` node publishes the retained `victron/ess/energy_today` message on every tick, so the first tick after deploy overwrote the saved totals with zeros before the `Reseed after restart` node could restore them.
- Fix: `Energy snapshot` now holds off publishing until `Reseed after restart` has run (stale or valid snapshot) or 30 s have passed. New flow variables `energyReseedDone`, `energySnapT0`.
- Tested offline: fresh deploy publishes nothing, reseed restores 16.27 / 20.44 kWh, next tick publishes the restored totals, fallback after 30 s, stale snapshot path.

## v16.4 - 2026-10-30
**Big-solar-day export (replaces the v16.3 per-tick PV trigger)**
- Problem: v16.3 re-evaluated PV every tick, so cloud passing over the array would switch export on and off and the battery would not be emptied in time for the ~14.5kW the arrays can charge at.
- Now the day is LATCHED. The first time PV exceeds `pvFullExportOnW` (8000W) on a day the forecast calls big (`isBigForecastDay`, forecast >= `bigDayForecastKwh`), the controller exports at `pvSurplusExportW` (-9000W) in EXPORT, NORMAL and PROTECT, whatever PV does afterwards.
- It stops when SoC falls under `pvSurplusExportMinSoc` (40%; re-arms at `pvSurplusExportResumeSoc`, 43%), when it is too late in the day (sunset minus `pvBigDayEndBeforeSunsetH`, else `pvBigDayEndFallbackHour` 16:30), or when PV stays under `pvBigDayLowPvW` (3000W) for `pvBigDayLowPvMin` (45 min), which is treated as a bust day. It re-arms if PV climbs back over 8000W.
- Needs a big forecast day, so a clear spell on an ordinary day will not trigger it.
- Never active in FAULT / FREECHARGE / DYNAMIC, with stale SoC, or before 08:00 (pre-dawn clamp wins).
- New CONFIG keys: `pvBigDayEndBeforeSunsetH`, `pvBigDayEndFallbackHour`, `pvBigDayLowPvW`, `pvBigDayLowPvMin` (plus the v16.3 keys, now meaning the big-day setpoint and SoC limits).
- `State publisher`: `pv.surplusExport` (export active) and `pv.bigSolarDayArmed` (day latched). Node status shows `BIG-DAY EXPORT` or `BIG-DAY armed`.
- Tested offline against 24 scenarios (arming, cloud dips, SoC cut-out and resume, end of day, bust day and re-arm, mode and stale-SoC guards, pre-dawn).

## v16.3 - 2026-10-30
**PV surplus export below the daytime floor**
- Problem: when SoC was under the daytime floor the mode was NORMAL/PROTECT, the setpoint stayed at -150W and all PV went into the battery even when the MPPTs out-produced what the inverter could use.
- New latching override in `ESS Controller`: if PV > `pvFullExportOnW` (8000W) in NORMAL/PROTECT, setpoint = `pvSurplusExportW` (-9000W). Releases when PV < `pvFullExportOffW` (7000W).
- Cuts out below `pvSurplusExportMinSoc` (40%); re-arms at `pvSurplusExportResumeSoc` (43%) to stop flapping.
- Never active in FAULT / FREECHARGE / DYNAMIC / EXPORT, with stale SoC, or before 08:00 (pre-dawn clamp wins).
- New CONFIG keys: `pvSurplusExportW`, `pvSurplusExportMinSoc`, `pvSurplusExportResumeSoc`.
- `State publisher`: `pv.fullExportOverride` now true for either override; new `pv.surplusExport`.
- Node status shows `PV-SURPLUS x.xkW`. Tab label and CONFIG header bumped to v16.3.
- Tested offline against 11 scenarios (engage/release/hysteresis/mode/pre-dawn).

## v16.1 - baseline
Flow as exported from Node-RED before version control was set up.
