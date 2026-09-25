# gcode_macro.cfg — change log

QIDI Q1 Pro, BIQU MicroProbe conversion. Baseline is the config as Klipper
parsed it on **2026-09-19** (pre-MicroProbe); current state is **2026-09-25**.

Derived by diffing `gcode_macro_CURRENT_2026-09-25.cfg` against the config dump
in `klippy.log`, not from memory.

**Totals:** 2 sections added · 7 modified · 5 removed · 41 unchanged

Backups live in `config-backups/`:

| file | what |
|---|---|
| `gcode_macro_2023-10-16_16-57-12.cfg` | QIDI factory, 2023-10-11, 40 sections |
| `gcode_macro_pre-microprobe_2026-09-19.cfg` | **drop-in revert target**, 53 sections |
| `gcode_macro_CURRENT_2026-09-25.cfg` | as-is today |
| `MERGED_pre-microprobe_2026-09-19.cfg` | all includes flattened — reference only, do NOT load |
| `printer_*`, `Adaptive_Mesh_*`, `plr_*`, `mesh_guard_*`, `timelapse_*` | supporting files |

---

## Added

### `[gcode_macro probe_deploy]` / `[gcode_macro probe_stow]`
```
probe_deploy:  SET_PIN PIN=probe_enable VALUE=0
probe_stow:    SET_PIN PIN=probe_enable VALUE=1
```
Drive the MicroProbe pin. Called from `[probe] activate_gcode` / `deactivate_gcode`
in `printer.cfg`. Verified correct by eye: pin extends on deploy, retracts on stow.

---

## Modified

### `[gcode_macro PRINT_START]` — restructured
- `set_zoffset` removed (macro no longer exists)
- Nozzle pre-heats to `hotendtemp - 80` instead of `M104 S0`, so probing happens warm but not oozing
- Optional `SOAK` parameter added (seconds of bed soak before homing)
- `M141` chamber set moved to the top; chamber wait added after `M109`
- **`VALIDATE_MESH` added after `G29`** (mesh guard)
- `CLEAR_NOZZLE` moved to *before* `M109`, restoring QIDI's original order so the
  macro's own cool-down isn't fighting the print-temp heat-up
- Redundant park block before `CLEAR_NOZZLE` removed (the macro parks itself; leaving
  it in triggered the `else` branch and added a spurious 5 mm lift)

### `[homing_override]` — Z branch replaced
Was: two `probe` calls sandwiched between `QIDI_PROBE_PIN_2` / `QIDI_PROBE_PIN_1`,
with hardcoded `SET_KINEMATIC_POSITION Z=1.9` and `Z=-0.1`.

Now:
```
G90
G1 X120 Y120 F7800
G28 Z
G1 Z30 F480
```
`G28` inside `homing_override` does not recurse, so this runs Klipper's real homing
against `probe:z_virtual_endstop` and applies the probe's `z_offset` properly.

Also removed the stray `QIDI_PROBE_PIN_2` before `G28 Z` in the all-axes branch and
the one on the macro's last line. Those were throwing `Unknown command` on every home.

### `[gcode_macro G29]` — partially updated ⚠️
`get_zoffset` calls removed from both branches.

**Still broken.** It retains the old normalisation:
```
G1 X{120 - probe.x_offset} Y{120 - probe.y_offset}
G1 Z10
probe                 <- reference probe
save_meshoffset
BED_MESH_CALIBRATE PROFILE=kamp
set_meshoffset        <- subtracts the reference a SECOND time
```
Klipper's `bed_mesh` already subtracts `probe.z_offset`. Under `[qdprobe]` that was
`0.000001` so it didn't matter; with a real `z_offset` of 1.53 the correction is
applied twice and the mesh comes out uniformly **−1.53**, which would drive the
nozzle into the plate. Caught by the mesh guard on 09-25.

Fix — delete the reference-probe block and both meshoffset calls:
```
BED_MESH_CLEAR
G28
BED_MESH_CALIBRATE PROFILE=kamp
SAVE_VARIABLE VARIABLE=profile_name VALUE='"kamp"'
SAVE_CONFIG_QD
```

### `[gcode_macro CLEAR_NOZZLE]` — rewritten
- `HOTEND` given a default of 250 (was required)
- `G90` added before the Z35 lift
- Purge cut from **80 mm to 30 mm**
- Wipe passes converted to a `{% for %}` loop, 5 iterations
- Wipe travel changed from X85↔X97 to **X77↔X97** (clears the MicroProbe)
- `M106 P2 S0` removed in two places
- **Second cooling stage removed** — the original finished with
  `G1 Y120 / G1 X230 / TEMPERATURE_WAIT MAXIMUM=140`, cooling to 140 °C at a
  different position. That's gone, so the blob is softer at the end than it used to be.
- `M104 S0` + `TEMPERATURE_WAIT MAXIMUM={hotendtemp-20}` retained (these are what
  actually make the wipe work — the heater is *off*, so 220 is where cooling starts,
  not where it ends)
- A `G4 P10000` dwell is present but **commented out** on that line

Known outstanding: no retraction after the purge, so PETG oozes after the wipe.
Optional fix — `G92 E0` then `G1 E-2 F1800` after the purge.

### `[gcode_macro CLEAR_NOZZLE_PLR]` — same treatment
Power-loss-recovery clean, invoked from the generated `.plr/plr.gcode`, not from config.
- Wipes converted to a `{% for %}` loop, 6 iterations
- `M400` + `M104 S0` + `TEMPERATURE_WAIT MAXIMUM={hotendtemp-20}` added
- Keeps the 80 mm purge, which is right here — filament has sat cold in a hot nozzle
- Still has **no `M107`**, so it exits with the part fan at full

### `[gcode_macro M191]` — cosmetic
Trailing `#MAXIMUM={s+1}` comment added. No behaviour change.

### `[gcode_macro RESPOND_INFO]` — cosmetic
`variable_s` renamed to `variable_S`. Klipper lowercases option names, so no
behaviour change, but it's an unnecessary diff.

---

## Removed

`get_zoffset` · `move_subzoffset` · `save_zoffset` · `set_zoffset` · `test_zoffset`

The entire nozzle-contact zero suite. Z=0 now comes from `[probe] z_offset` instead
of being re-measured against the plate every print.

**Consequences:**
- `z_offset` is now **nozzle-specific**. Any nozzle change, reseat, or deposit shifts
  the first layer silently. Re-run calibration after any nozzle work.
- The drift detector is gone. A 0.17 mm blob used to show up as a number in the log;
  now it just ruins prints.
- `PRINT_END` still calls `SAVE_ZOFFSET` → `Unknown command` on every print. Remove it.
- `test_zoffset` was the manual gap check. Gone.

---

## Related changes in `printer.cfg`

| change | note |
|---|---|
| `[qdprobe]` → `[probe]` | pin `^!gpio21`, x_offset 24, y_offset 10 |
| `[heater_fan hotend_fan2]` **removed** | header was unused; gpio11 freed for `[output_pin probe_enable]` |
| `[stepper_z] position_endstop` removed | correct for a virtual endstop |
| `z_offset` set by hand at line 434 | `SAVE_CONFIG` will not persist it — see below |

---

## Open issues

1. ~~`hotend_fan2`~~ — **resolved 2026-09-25.** The header was never populated on this
   machine, which is why gpio11 was free to repurpose for `probe_enable`. No fan lost.
2. ~~`PRINT_END` calls `SAVE_ZOFFSET`~~ — **resolved 2026-09-25**, call removed.
3. **`G29` double-corrects the mesh.** `save_meshoffset` captures the reference probe
   (1.5648) and `set_meshoffset` runs `ADD_Z_OFFSET_TO_BED_MESH ZOFFSET={0 - zoffset}`,
   shifting the whole mesh by −1.5648 on top of the subtraction `BED_MESH_CALIBRATE`
   already did. Fix: delete the reference-probe block and both meshoffset *calls*
   (keep the macro definitions). Blocks printing.
4. **`SAVE_CONFIG` does not persist `[probe] z_offset`** on this fork — three
   calibrations (−1.030, 1.950, 1.530) all wrote `0.000`. Set it by hand at
   `printer.cfg` line 434 and keep the `#*# [probe]` block deleted.
   `SAVE_CONFIG_QD` during a print does *not* clobber it — verified.
5. **Probe mount geometry.** Pin stroke is ~1.95 mm and that's the whole budget:
   `deployed reach + stowed clearance = 1.95`. Currently 1.53 / 0.42 at a 2.75 mm
   sanded plate. Target an even split — raise the probe seat ~0.75 mm from the
   3 mm design for ~1.0 / ~0.95.
6. **Restoring the piezo.** The MicroProbe went onto the old hall-sensor wiring, so
   the piezo is still connected. `[qdprobe]` registers as `probe` and provides the
   PIN_1/PIN_2 mux, so reverting to it would give both sensors — MicroProbe for mesh,
   piezo for the nozzle zero. Blockers: `[qdprobe]` has no `activate_gcode` for
   deploy/stow, and gpio11 is contested with `hotend_fan2`.
