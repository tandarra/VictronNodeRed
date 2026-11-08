# Changelog

Newest first. Each entry = one commit + tag; `git show <tag>:flows.json` restores that version.

## v16.10 - 2026-11-09
**Minimum SoC lowered from 20% to 15%**
- `CONFIG.minSocNormal` 20 -> 15. This is the ESS "Minimum SOC" the controller writes to the Cerbo in every mode except FREECHARGE (which still writes 100%). Below this SoC the Victron ESS stops discharging and the house imports from the grid, so the battery now carries the house 5 points (about 1.4 kWh of the 28.68 kWh pack) further before importing.
- Same value feeds the Trajectory Engine and Forecast Adjuster floors and the `Battery time calc` "to X%" estimate, so those now count down to 15% too. The `|| 20` fallbacks in those nodes are now `|| 15`.
- Unchanged: daytime floors, big-day tiers (25 / 30%), the weather hold's 60% export floor, the 19:00 / 20:00 shed and the 43% 21:00 target.

## v16.9 - 2026-11-06
**No exporting below 60% SoC while rain / thunderstorm is forecast**
- While the weather hold applies (rain, heavy rain or thunderstorm forecast before 19:00) the battery is no longer exported below `weatherImpact.minSocPct` (60%): at or below 60% the setpoint goes to -150 W, and export only resumes once SoC climbs back to `resumeSocPct` (62%), so the setpoint doesn't flap around the line. This matters on days when the dynamic floor is low (e.g. big forecast days, floor 25-35%), so rain/storm can no longer pull the pack under 60% when PV may not refill it for the evening.
- Only the EXPORT is stopped. MinSoC is deliberately NOT raised: a MinSoC above the current SoC makes the Victron ESS grid-charge the battery (seen with BALANCE mode earlier). The house load is still served from the battery, so SoC can still drift below 60% under load with no PV.
- The latch resets when the hold ends (weather clears or 19:00). Dry days, missing weather data, DYNAMIC (19:00 / 20:00 shed), FREECHARGE and FAULT are unaffected.
- Node status shows `WX HOLD <label> cap <W> (SoC floor 60%)`; `state.weatherHold.atMinSoc` is published.
- Tested offline with a 35% dynamic floor: rain at 75% and 61% exports at -2000 W, stops at 60%, stays stopped at 61% (latched), resumes at 62.5%; storm stays at -150 W; rain plus an armed big-day latch at 58% gives -150 W; dry still exports -9500 W; latch cleared when the weather clears.

## v16.8 - 2026-11-06
**Rain / thunderstorm now HOLD the daytime export instead of changing the targets**
- Design change after v16.6/v16.7: the existing 19:00 (65% checkpoint) and 20:00 (21:00 target) evening shed already brings SoC down, so the weather no longer touches the 21:00 target or the daytime floor. Both bumps are removed and `Forecast Adjuster` is back to its original logic with the 43% absolute minimum (`forecastMinTarget` 43). The target is 43% in every condition.
- New behaviour: every forecast refresh (15 min) the `Weather impact` node scores the weather from now until `CONFIG.weatherImpact.holdUntilH` (19:00). While it sees rain, heavy rain or a thunderstorm, `ESS Controller` caps the export setpoint: rain `-2000 W` (slow), heavy rain and thunderstorm `-150 W` (halt) - `weatherImpact.exportCapW`. The battery is preserved for the afternoon and dinner.
- The hold also switches off the big-day latch (-9000 W) and the PV full-export override while it applies. It only touches EXPORT / NORMAL / PROTECT; FREECHARGE, DYNAMIC (the 19:00 / 20:00 shed) and FAULT are never limited. At 19:00 the hold ends and the evening shed takes over. It lifts by itself when the forecast clears, and is ignored if the weather data is older than 3 h (e.g. DNS failure).
- Weather after 19:00 is ignored (the shed owns that period). Detection uses the HOURLY weather codes, precipitation (mm) and probability, so a storm that has already passed does not trigger it.
- Removed from v16.7: the 5-minute latch release (`pvBigDayStormLowPvMin`), since the hold covers it and resumes the latch automatically when the weather clears.
- `State publisher`: `state.weatherImpact` (severity 0-3) and `state.weatherHold` (`{label, capW}` or false). Node status shows `WX HOLD <label> cap <W>`.
- Tested offline: dry / no data unchanged (-9500), rain -> -2000, heavy rain and thunderstorm -> -150, big-day latch overridden while storm is forecast but back to -9000 with no weather, hold on at 18:59 and released at 19:00, DYNAMIC (19:30, 20:30) and FREECHARGE untouched, stale weather ignored, 21:00 target 43% at every forecast size.

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
