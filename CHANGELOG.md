# Changelog

Newest first. Each entry = one commit + tag; `git show <tag>:flows.json` restores that version.

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
