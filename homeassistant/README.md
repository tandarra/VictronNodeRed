# Home Assistant side (pool control)

The pool is controlled from Home Assistant, driven by the ESS state this flow publishes on `victron/ess/state`.
The full package (`packages/pool_solar.yaml`) lives in the Home Assistant config repo; this folder documents
what it does and keeps the automations and the changes made alongside this flow.

## How the pool controller decides (runs every 2 min)
Pick one mode, first match wins:
1. **Shed** - `input_boolean.pool_pump_shed` on: mode `off` (heat pump stops, pump runs on 3 min, then stops).
2. **Free window** - Mon-Fri 21:00-23:55: heat to the night target on free power.
3. **Solar** - daytime heating to the day target (30 C).
4. **Filter** - pump only, until the daily minimum (6 h) is met.
5. **Off.**

### SoC gate, PV-surplus bypass and late run (added 2026-10-30)
- Normal rule: start at the SoC window's start level (85 % on a big day), pause below its pause level (80 %).
- **PV-surplus bypass (big solar day):** from the big-day start (10:00) the SoC gate is ignored while `sensor.ess_pv_power`
  >= `pool_solar_pv_start` (6000 W), as long as SoC >= `pool_solar_soc_min_surplus` (35 %). Once running it holds until PV
  < start minus `pool_solar_pv_margin` (1500 W, i.e. 4500 W).
  Reason: the MPPTs can charge more than 14 kW while the inverter passes about 8 kW, so a slower battery fill costs nothing.
- **Late run:** past the 16:30 cutoff (`pool_day_cutoff`) a big day carries on while SoC >= the pause level (80 %). After 18:00 it
  stops when PV has been under 2 kW for 2 min (`binary_sensor.pool_pv_below_2_kw`). Whichever comes first.

### Load shed
`pool_pump_shed_automation.yaml`: AC loads > 8 kW for 10 s sets the flag, < 5 kW for 3 min clears it. Skipped Mon-Fri 21:00-23:59.

## Changes made to `packages/pool_solar.yaml`
1. Header comment documents the load shed.
2. Controller triggers include `input_boolean.pool_pump_shed` and `binary_sensor.pool_pv_below_2_kw`.
3. New controller variables:
   `shed`, `pv_w`, `pv_start`, `pv_margin`, `surplus_floor`, `surplus`, `late_ok`.
4. `desired` starts with `{% if shed %}off`.
5. `want_solar` / `want_filter` accept `(soc >= pause or surplus)` / `(soc >= resume or surplus)`.
6. `daytime` end condition: `now() < pool_day_cutoff or late_ok`. Status sensor uses the same rule.
7. New template binary sensor `Pool PV below 2 kW` (`delay_on: 00:02:00`, state `sensor.ess_pv_power < 2000`).

## Related Node-RED fields (v16.4+)
`victron/ess/state` also carries `pv.surplusExport` (big-day export active) and `pv.bigSolarDayArmed` (day latched).
They are not yet wired to HA sensors.
