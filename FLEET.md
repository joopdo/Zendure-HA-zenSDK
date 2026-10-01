# Zendure Fleet — second SolarFlow 800 Plus alongside Gielz1986 zenSDK

> ## ⚠️ Day-1 requirement — do this BEFORE enabling the layer
> Set the vendor's `input_number.zendure_setting_max_charge_power` **and**
> `input_number.zendure_setting_max_discharge_power` to **800 W**.
> Unit 1 must never try to serve more than itself — everything above 800 W is
> unit 2's job now. Skipping this makes the vendor loop chase power it cannot
> deliver and distorts the fleet split.

Two HA packages that add a second 800 Plus **without touching** the vendor
package or its automation (verified against vendor `v20260824`).

| File | Role |
|---|---|
| `packages/zendure2.yaml` | Unit 2 I/O: IP helper, REST sensors, REST commands, RAM-mode + idle-standby housekeeping |
| `packages/zendure_fleet.yaml` | Coordinator: `sensor.zendure2_target_power` (the brain), the control loop, mode/dynamic mirror, fleet sensors, vendor-contract watchdog |

See `MAINTAINING.md` for the fork/upstream-sync workflow.

## Control architecture

**Unit 1 (vendor, unchanged) = fast regulator.** Its 5s loop keeps regulating
P1 toward 0 within 0–800 W, exactly as today.

**Unit 2 (coordinator) = slow baseload setter.** Every `zendure2_loop_interval_s`
(default/floor 20 s — 4× slower than the vendor loop, deliberately) it computes
the *absolute* fleet target `(z1 + z2 − P1)` — total power the fleet should
deliver/absorb — takes unit 2's SoC-weighted share, and commands only unit 2.
Because P1 already reflects unit 2's output, unit 1's loop automatically
settles on the remainder. Time-scale separation plus the write deadband
(`zendure2_write_deadband_w`, default/floor 25 W) and stop-before-reverse keep
the loops from fighting.

Split rules (`sensor.zendure2_target_power`, fully observable in HA):
- **|fleet target| ≤ single-unit limit (default 700 W):** one unit runs.
  Unit 1 by default; unit 2 takes over completely when its SoC advantage
  exceeds the lead band (default 8%) — discharging when fuller, charging when
  emptier. Handover is automatic (target→0, unit 1 picks up the residual).
- **Above the limit:** proportional split by usable headroom
  (`soc − min_soc` for discharge, `max_soc − soc` for charge), per-unit clamp
  0–800 W with spill-over to the other unit.
- SoC/limit guards per unit; unit 2 covers everything when unit 1 is
  SoC-limited, and vice versa.

Mode handling:
- **Smart Matching family** → the coordinator loop above. "(Expensive)"
  discharges only while `sensor.dynamic_highest_price_period == Yes` + spread
  OK, mirroring the vendor's own conditions.
- **Dynamic Trading windows** → event mirror on the vendor's
  `input_boolean.dynamic_charging_running` / `dynamic_discharging_running`
  flags: unit 2 charges/discharges at its cap for exactly the vendor's window.
  No Nordpool logic duplicated.
- **Quick Charge/Discharge** → unit 2 joins at its cap on mode change.
- **Manual** → unit 2 takes the overflow above 800 W of
  `input_number.zendure_manual_power`.
- **Standby** → unit 2 goes to full standby.

## Safe bring-up (observe before control)

Two switches gate every write to unit 2; both are created **OFF** by HA, so a
fresh install behaves exactly like the single-battery setup until you act:

- `input_boolean.zendure2_enabled` — master kill switch. While OFF, no
  automation in this layer sends anything to unit 2.
- `input_boolean.zendure2_dry_run` — observe mode. While ON, the coordinator
  computes `sensor.zendure2_target_power` and writes a logbook entry for every
  action it *would* take (loop writes and mode/window mirrors), but sends no
  REST commands. (HA cannot default a toggle to ON without re-forcing it on
  every restart — which would silently disable live control in production — so
  dry run is OFF at creation and the enabled switch provides first-load safety
  instead. Follow the order below.)

Bring-up sequence:
1. Day-1 requirement above (vendor max powers → 800 W).
2. Configure unit 2 (steps below), leave `zendure2_enabled` OFF.
3. Turn `zendure2_dry_run` **ON**, then `zendure2_enabled` **ON**.
4. Run a day or two: compare the 🧪 DRY RUN logbook entries and
   `sensor.zendure2_target_power` against real P1 behavior
   (`sensor.zendure2_applied_power` stays 0 — nothing is written).
5. When the intended splits look right, turn `zendure2_dry_run` **OFF**.
   Unit 2 is now live.

## Install

1. Keep the vendor package + automation exactly as they are (this fork carries
   them unmodified except the nordpool string shim, see `MAINTAINING.md`).
2. Copy `packages/zendure2.yaml` and `packages/zendure_fleet.yaml` into
   `config/packages/` (packages must be enabled under
   `homeassistant: packages:`). Restart HA.
3. Collision check (one-time): the new layer only creates `zendure2_*` and
   `zendure_fleet_*` names — verified against the vendor package. Confirm your
   other local packages don't already use those prefixes:
   ```sh
   grep -rn "zendure2_\|zendure_fleet_" config/packages/ \
     --include="*.yaml" -l | grep -v "zendure2.yaml\|zendure_fleet.yaml"
   ```
   No output = no collisions.
4. Set `input_text.zendure2_setting_ip_address` to unit 2's IP (give it a DHCP
   reservation).
5. Helpers (fresh installs start at the value in parentheses):
   - `zendure2_setting_max_charge_power` / `..._discharge_power` → 800 (0)
   - `zendure_fleet_soc_lead_band` → 8 (2)
   - `zendure_fleet_single_unit_limit` → 700 (100)
   - `zendure2_loop_interval_s` → 20 (20, also the floor)
   - `zendure2_write_deadband_w` → 25 (25, also the floor)
6. Follow the bring-up sequence above.

## Surviving a Gielz version bump

- Vendor files stay pristine (one isolated shim commit — see `MAINTAINING.md`
  for the tag-by-tag upstream merge workflow).
- This layer only reads the vendor entities listed at the top of
  `zendure_fleet.yaml` (mode select, dynamic flags, settings helpers, a handful
  of unit-1 sensors). These are the package's UI-facing core and rarely change.
- `binary_sensor.zendure_fleet_vendor_contract_ok` re-checks that list
  continuously; if an update renames something it flips off, the `missing`
  attribute names the culprit, and the watchdog automation sends a persistent
  notification. Until fixed, unit 2 simply idles — unit 1 keeps working.

## Fleet sensors

- `sensor.zendure_fleet_power` — combined signed battery power (+charge/−discharge)
- `sensor.zendure_fleet_state_of_charge` — capacity-weighted SoC
- `sensor.zendure_fleet_capacity` — combined kWh
- `sensor.zendure_fleet_available_energy` — usable kWh above min SoC
- `sensor.zendure_fleet_required_energy` — kWh of charge headroom to max SoC

## Tuning

- Oscillation at the 1-unit/2-unit boundary → raise `zendure2_write_deadband_w`
  or `zendure2_loop_interval_s`.
- Units ping-ponging lead at low power → widen `zendure_fleet_soc_lead_band`.
- Second unit engaging too late in evening peaks → lower
  `zendure_fleet_single_unit_limit` (e.g. 500 W).
