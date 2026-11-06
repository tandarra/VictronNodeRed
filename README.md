# VictronNodeRed

![Victron ESS Controller](docs/hero-overview.png)

Node-RED ESS controller for the Tandarra Victron system at Pokeno, NZ, with the Home Assistant pool automation that follows it.

**[Live view (embedded VRM)](https://tandarra.github.io/VictronNodeRed/)** · [Open in VRM](https://vrm.victronenergy.com/installation/515169/share/20610da3) (read-only)

| | |
|---|---|
| Inverter / charger | MultiPlus-II 48/10000 (about 8 kW AC) |
| Solar | 19.44 kW across 3 arrays, 5 MPPTs (can charge a battery at more than 14 kW) |
| Battery | 28.7 kWh LFP |
| Controller | Cerbo GX running this Node-RED flow |
| Home Assistant | pool control, dashboards, energy costs |

## What the flow does
![Flow editor](docs/hero-flow.png)

A 60 s tick runs: phase logic, forecast adjuster (Open-Meteo, per-array model), trajectory engine, safety gate, then the **ESS Controller** writes the grid setpoint and minimum SoC to the Cerbo and the **State publisher** sends the full state to MQTT (`victron/ess/state`).

- **Forecast-driven export and overnight charging** on cheap/free power.
- **Big solar day export (v16.4):** the first time PV passes 8 kW on a big forecast day the day is latched and the setpoint goes to -9000 W, so the battery empties into the grid while the MPPTs run flat out. Stops below 40 % SoC (resumes at 43 %), at sunset minus 3 h (16:30 fallback), or if PV stays under 3 kW for 45 min (bust day). Never overrides FAULT, free charge, stale SoC or the pre-08:00 clamp.
- **Weather export hold (v16.8):** when the forecast (refreshed every 15 min) shows rain before 19:00 the daytime export is slowed (-2000 W); heavy rain or a thunderstorm halts it (-150 W) and also stops the big-day latch, so the battery lasts the afternoon and dinner. At 19:00 the existing evening shed takes over and brings SoC down to the 21:00 target (43% minimum). Tunable in `CONFIG.weatherImpact`.
- **Daily energy counters** (grid import/export, AC load) survive a redeploy (v16.5).

![Cerbo GX](docs/hero-cerbo.png)

## Pool control (Home Assistant)
See [`homeassistant/`](homeassistant/README.md): solar heating with a PV-surplus bypass so the pool starts at 10:00 on big days, a late run to 18:00+, and a load shed that stops the pump when AC loads go above 8 kW.

## Install
Node-RED: Menu > Import > select `flows.json` (replace the existing tab) > Deploy.

## Versions
See [`CHANGELOG.md`](CHANGELOG.md).

| Version | Change |
|---|---|
| v16.8 | Rain / thunderstorm forecast holds daytime export until the 19:00 evening shed (supersedes v16.6 bumps and v16.7 latch release) |
| v16.7 | 21:00 target never below 43%; big-day export stops within 5 min when a storm collapses PV |
| v16.6 | Rain / thunderstorm raise the 21:00 target and daytime floor |
| v16.5 | Redeploy no longer wipes the daily energy counters |
| v16.4 | Big-solar-day export latch (-9000 W) |
| v16.3 | PV surplus export (superseded by v16.4) |
| v16.1 | Baseline |

## Rollback
Every update is one commit and one tag. To get an older flow:

    git log --oneline -- flows.json
    git show <commit-or-tag>:flows.json > flows-old.json

Then import `flows-old.json` into Node-RED.
