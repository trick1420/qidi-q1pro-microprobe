# Change reference

Every change from the stock QIDI Q1 Pro configuration, and the reason for each. The
`README.md` file explains how to make these changes. This file records what they are, so
you can compare against your own machine or undo one of them.

Baseline is the configuration as QIDI shipped it, captured on 2026-09-19 before the
MicroProbe conversion. Current state is 2026-09-25.

Stock and pre-conversion copies of every file are in `config-backups/`.

## printer.cfg

### `[smart_effector]` — the piezo sensor

| Setting | Stock | Now | Why |
|---|---|---|---|
| `x_offset` | `17.6` | `24` | Klipper applies one pair of offsets to whichever sensor is active. Only the MicroProbe probes the mesh, so these must be its offsets. Measure your own mount. |
| `y_offset` | `4.4` | `10` | Same reason. |

`pin: U_1:PC1` and `z_offset: 0.000001` do not change. The piezo wiring is untouched by
this conversion.

### `[qdprobe]` — now the MicroProbe

| Setting | Stock | Now | Why |
|---|---|---|---|
| `pin` | `!gpio21` | `^!gpio21` | The MicroProbe needs the internal pull-up resistor. The inductive sensor did not. |
| `activate_gcode` | absent | `probe_deploy` then `G4 P500` | Extends the pin before probing. The stock sensors had no moving parts. |
| `deactivate_gcode` | absent | `probe_stow` | Retracts the pin afterwards. |
| `deactivate_on_each_sample` | absent | `false` | Deploys once per probing run instead of once per sample. |

`z_offset: 0.000001` does not change, and must not. `[qdprobe]` and `[smart_effector]`
share one probe object, so a real value here gets applied twice and drives the nozzle
into the plate.

### `[heater_fan hotend_fan2]` — removed

Stock declares a second hotend heatsink fan on `gpio11`. A stock Q1 Pro has only one
heatsink fan fitted, so the section drives nothing. Removing it frees `gpio11` for the
MicroProbe control pin, and Klipper refuses to start if two sections claim one pin.

If you added a second hotend fan yourself, keep this section and pick a different pin for
`probe_enable`.

### `[output_pin probe_enable]` — new

```ini
[output_pin probe_enable]
pin: gpio11
value: 1
```

Controls the MicroProbe pin. `value: 1` starts the pin retracted, which the piezo needs.

### `[bed_mesh]`

`vibrate_gcode: Z_DOUDONG` is commented out. `Z_DOUDONG` conditions the piezo sensor
before it measures. The MicroProbe probes the mesh and does not need it.

### `[stepper_z]`

`position_endstop` changed from `-0.2` to `-0.23`. This sets the Z frame between `G28`
and `get_zoffset`, and `get_zoffset` overwrites it a moment later, so the value has no
effect on printing. Klipper refuses to start without the line.

### Includes

`[include mesh_guard.cfg]` added, above the `#*# SAVE_CONFIG` marker. Anything below that
marker is rewritten automatically, on every print.

## gcode_macro.cfg

### New macros

```ini
[gcode_macro probe_deploy]
gcode:
    SET_PIN PIN=probe_enable VALUE=0

[gcode_macro probe_stow]
gcode:
    SET_PIN PIN=probe_enable VALUE=1
```

Swap the two `VALUE` numbers if your pin moves the wrong way.

### `CLEAR_NOZZLE`

| Change | Why |
|---|---|
| Wipe travel `X85` and `X65` becomes `X77` | `X65` collides with the MicroProbe |
| Purge cut from `80 mm` to `30 mm` | Enough to clear ooze on a nozzle that is already clean |
| `G92 E0` then `G1 E-2 F1800` after the purge | Relieves melt-zone pressure. Without it, PETG oozes after the wipe and forms a curl on the tip. |
| `G4 P5000` before the first wipe | Lets the purge blob set so it snaps off rather than smearing |
| First wipe at `180 °C`, second wipe at `140 °C` | Stock waits only for a `20 °C` drop, so it wipes at `230 °C` while PETG still flows, then oozes on the way down with nothing to catch it. The second pass removes that ooze. |

`CLEAR_NOZZLE_PLR` gets the same treatment and an `M107` at the end. Stock leaves the
part fan at full. It runs only when you resume after a power cut.

### `PRINT_START`

Restructured. `CLEAR_NOZZLE` runs before `G29` rather than after `M109`, so the nozzle
cools during meshing instead of making you wait. `VALIDATE_MESH` runs immediately after
`G29`. An optional `SOAK` parameter adds a bed soak in seconds.

### `G29`

The `G28` call is now conditional:

```ini
{% if 'xyz' not in printer.toolhead.homed_axes %}
    G28
{% endif %}
```

`PRINT_START` already homes, so stock homes twice per print. The condition removes the
duplicate and still lets you run `G29` on its own.

### Unchanged

`get_zoffset`, `move_subzoffset`, `set_zoffset`, `save_zoffset`, `test_zoffset`,
`homing_override`, `set_meshoffset`, `save_meshoffset`, and `Z_DOUDONG` are all stock.

Do not delete them. They are the piezo measurement path, and they work unmodified because
`[qdprobe]` still provides `QIDI_PROBE_PIN_1` and `QIDI_PROBE_PIN_2`.

## Adaptive_Mesh.cfg

| Line | Setting | Stock | Now |
|---|---|---|---|
| 24 | `variable_margin_enable` | `False` | `True` |
| 25 | `variable_margin_size` | `5` | `15` |
| 168 | `variable_distance_to_object_y` | `10` | `5` |

KAMP probes only the area your part occupies and draws the purge line outside it, so the
purge prints on ground the printer never measured. These three settings extend the mesh
15 mm past the part and move the purge to 5 mm from it.

Both of the first two are required. Line 62 forces `margin_size` to `0` whenever
`margin_enable` is `False`.

## New files

`mesh_guard.cfg` reads the bed mesh after `G29` and stops the print if the numbers are
implausible. A healthy mesh on this machine has a range of `0.03` to `0.17 mm`. One probe
misfire produced a range of `1.175 mm` and a correction that lifted the nozzle `0.954 mm`
off the plate.

Default limits are range `0.80`, peak `0.60`, and step `0.25 mm`. A full-bed mesh
legitimately reaches a peak of about `0.45`, so do not tighten these below `0.5`.

## Measured values from the test machine

Your numbers will differ. These show what a healthy result looks like.

| Value | Reading | Where it comes from |
|---|---|---|
| MicroProbe standoff | `1.601 mm` | `Result is z=` from `get_zoffset`, every print |
| Stowed pin clearance | `0.76 mm` | Feeler gauge under the pin at `Z0`, minus `0.07` |
| Pin travel | `2.36 mm` | Standoff plus clearance |
| Bed mesh range | `0.110 mm` | `MESH CHECK` console line |
| Bed mesh peak | `0.083 mm` | `MESH CHECK` console line |
| Piezo sample spread | `0.007 mm` | Five samples per `get_zoffset` run |

Watch the standoff. It is measured on every print and written to `klippy.log`. A drop of
more than about `0.1 mm` means something is on the nozzle tip.
