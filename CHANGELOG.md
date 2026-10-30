# Changelog

Newest first. Each entry = one commit + tag; `git show <tag>:flows.json` restores that version.

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
